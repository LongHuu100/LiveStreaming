## Từ Av inputs to server --> Html5
B1: Server public một rtmp enpoint để máy quay phát video trên endpoint này.

B2: Server chuyển đổi rtml sang hls định dạng file m3u8 chứa các luồng video file .ts

B3: Trình duyệt mở file m3u8 này để phát live streaming.

## Thủ tục kết nối gồm 7 bước như sau.
<img src="https://docs.flashphoner.com/download/attachments/9242082/rtmp_streaming_call_flow.jpg?version=1&modificationDate=1548294460000&api=v2" />

## ThreadPool
``` text
Trong hàm main mai.cpp gọi EventPollerPool::setPoolSize(threads); để set số threads được chạy
Khi gọi lần đầu _poller = EventPollerPool::Instance().getPoller() sẽ thực hiện 2 bước:
B1: Tạo các poller như sau: 
Sẽ khởi tạo một: EventPollerPool::EventPollerPool() {
     auto size = addPoller(....);
}
// Trong addPoller có vòng lặp để tạo thread như sau.
for (size_t i = 0; i < size; ++i) {
     Tạo ra các EventPoller
     EventPoller::Ptr poller(new EventPoller((ThreadPool::Priority) priority));
     poller->runLoop(false, register_thread);
     Trong runLoop có một _loop_thread = new thread(&EventPoller::runLoop, this, true, ref_self); sẽ tạo thread và chạy
     vòng while để lấy callback trong _event_map và chạy chúng.
     .....
     Lưu các poller vào biến chứa các thread để sau này lấy ra chạy ở bước 2.
     _threads.emplace_back(std::move(poller));
     .....
     Như vậy là chỉ cần đẩy callback vào trong _event_map là thread lấy trong _event_map
     để chạy task. VD như:
     _socket->setOnAccept([this](Socket::Ptr &sock, shared_ptr<void> &complete) {
        auto ptr = sock->getPoller().get();
        auto server = getServer(ptr);
        // Gọi tới async để đẩy callback function vào trong _event_map
        // [server, sock, complete] là captrue các biến server, sock, complete vào trong callback.
        ptr->async([server, sock, complete]() {
            server->onAcceptConnection(sock);
        });
    });
}
Đến đây là đã có _threads chứa các thread chạy vòng lặp lấy cb trong _event_map.

B2: Lấy thread để add task
EventPoller::Ptr EventPollerPool::getPoller(bool prefer_current_thread) {
    auto poller = EventPoller::getCurrentPoller();
    if (prefer_current_thread && _prefer_current_thread && poller) {
        return poller;
    }
    return dynamic_pointer_cast<EventPoller>(getExecutor());
}
getExecutor() sẽ lấy ra một thread đã có trong bước 1.

===> Như vậy là khi gọi EventPollerPool::getPoller sẽ gọi ra một thread trong polls
     Sau đó sẽ thêm callback vào _event_map của thread đó
     Thread chạy vòng lặp while để lấy task trong _event_map chạy. 

#### Đọc và Ghi message xuống socket.
Hàm TcpServer::start_l có: 
     _socket->listen(...);
Socket::listen(uint16_t port, const string &local_ip, int backlog) Gọi tới
Socket::listen(const SockFD::Ptr &sock), trong hàm này có:
     strong_self->onAccept(strong_sock, event);
Hàm onAccept có
if (!peer_sock->attachEvent(peer_sock_fd, false)) {
     peer_sock->emitErr(SockException(Err_eof, "add event to poller failed when accept a socket"));
}
Trong attachEvent có:
     Add 3 lần cho 3 event Event_Read, Event_Write, Event_Error vào polls.
     int result = _poller->addEvent(sock->rawFd(), EventPoller::Event_Read | EventPoller::Event_Error | EventPoller::Event_Write

     if (event & EventPoller::Event_Read) {
          strong_self->onRead(strong_sock, is_udp);
     }
     if (event & EventPoller::Event_Write) {
          strong_self->onWriteAble(strong_sock);
     }
     ....
Trong onWriteAble có:
     flushData(sock, true);
Trong flushData có 
     decltype(_send_buf_sending) send_buf_sending_tmp;
     send_buf_sending_tmp.emplace_back(BufferList::create(...))
     while (!send_buf_sending_tmp.empty()) {
        auto &packet = send_buf_sending_tmp.front();
        auto n = packet->send(fd, _sock_flags);
        ....
     }
====> Như vậy là trong polls sẽ có những thread chỉ đọc, chỉ gửi, chỉ bắn lỗi.
     Khi muốn gửi message chỉ cần đẩy mesaage vào _send_buf_waiting là thread sẽ gửi.
     VD: RtmpSession.h có hàm:
     void onSendRawData(toolkit::Buffer::Ptr buffer) override{
        _total_bytes += buffer->size();
        // Hàm send sau một vài lệnh sẽ gọi tới Socket::send_l đẩy data vào _send_buf_waiting
        // và gọi flushData
        send(std::move(buffer));
     }
```

## Code Note.
``` C++
// Trong server/main.cpp sẻ khởi tại một tcp server.
auto rtmpSrv = std::make_shared<TcpServer>();
Chạy 2 hàm khởi tạo là TcpServer::TcpServer và từ lớp cha Server::Server
/* Trong TcpServer::TcpServer sẽ Set một callback cho _on_accept để sử dụng trong Socket::listen */
_socket->setOnAccept([this](Socket::Ptr &sock, shared_ptr<void> &complete) {
     auto ptr = sock->getPoller().get();
     auto server = getServer(ptr);
     ptr->async([server, sock, complete]() {
          /* Bên trong hàm này có hàm setOnRead sẽ tạo một _on_read là một callback để gọi trong 
          * Socket::listen --> strong_self->onAccept.
          * _on_accept(peer_sock, completed); peer_sock->attachEvent --> Socket::onRead
          * --> strong_session->onRecv(buf) chính là các implement như RtmpSession::onRecv ...
          */
          // Hàm này đồng thời gọi _session_alloc là một callback quản lý session của tcp được setup lúc gọi
          // rtmpSrv->start<RtmpSession>(rtmpPort)
          // Trong hàm này cũng lưu các session trong _session_map có chứa các Socket::Ptr &sock.
          // ===> Server đã có được các session kết nối đến để khi gửi message thì sẽ gửi đúng client.
          // Khi socket mất kết nối thì exception gọi strong_session->shutdown(ex) sẽ gọi vào Socket::setOnErr
          // trong _on_err là một callback được đẩy vào từ sock->setOnErr(....) sẽ xoá các session trong _session_map
          server->onAcceptConnection(sock);
     });
});
/* Start server với RtmpSession để quản lý các session kết nối đến TcpServer
* Trong RtmpSession sẽ gửi các lệnh điều khiển bên trên xuống các socket kết nối
*/
rtmpSrv->start<RtmpSession>(rtmpPort);
/* Gọi vào TcpServer::start_l --> Chạy vào Socket::listen để gọi _on_accept và _on_read là 2 callback đã set ở trên. */


// Public một enpoint để máy quay thả luồng video vào đây, có thể là rtmp hoặc rtsp
PlayerProxy::Ptr player(new PlayerProxy("app","stream"));
player->play("rtmp://live.hkstv.hk.lxdns.com/live/hks");
Gọi đến: startConnect(host_url, port, play_timeout_sec) --> TcpClient::startConnect
Tại đây gọi đến TcpClient::onSockConnect(...) 
     try {
          strong_self->onRecv(pBuf) chính là RtmpSession::onRecv;
          -- onParseRtmp
          -- input
          -- onSearchPacketTail(...) -> _next_step_func để trả data vào các callback đã set trước đó.
               handle_S0S1S2 --> handle_rtmp --> handle_chunk --> onRtmpChunk...
     } catch (std::exception &ex) {
          strong_self->shutdown(SockException(Err_other, ex.what()));
     }
     - Tiếp theo thả event onConnect(ex) xuống RtmpPlayer::onConnect(..);
Tại RtmpPlayer::onConnect gọi đến RtmpProtocol::startClientSession
--- RtmpProtocol::handle_rtmp
--- RtmpProtocol::handle_chunk
--- onRtmpChunk(std::move(packet)): hàm này của RtmpPlayer::onRtmpChunk.
--- RtmpPlayer::onMediaData_l
--- RtmpPlayerImp.h onMediaData(..) --> _demuxer->inputRtmp(chunkData);
Tại RtmpDemuxer::inputRtmp:
     Lần đầu chưa có track thì gọi makeVideoTrack(..) set được
          _video_track = H264Track;
          _video_rtmp_decoder = H264RtmpEncoder;
          _video_rtmp_decoder->addDelegate(_video_track);
          addTrack(_audio_track); Chính là Demuxer::addTrack chi tiết ở [My_Note_Track]

     Lần sau đã có track thì:
          _video_rtmp_decoder->inputRtmp(pkt);
          Gọi vào H264RtmpEncoder::inputRtmp --> H264RtmpDecoder::onGetH264 --> Frame::inputFrame;
          Thả frame vào các delegate đã được add vào từ bước "Lần đầu chưa có track" 
          for (auto &pr : _delegates) {
               if (pr.second->inputFrame(frame)) {
                    ret = true;
               }
          }

/* [My_Note_Track]: Trong hàm _sink->addTrack sẽ cho track thêm một addDelegate để lắng nghe các Frame của track */
if (_sink->addTrack(track)) {
     /* track->addDelegate là FrameDispatcher.addDelegate */
     track->addDelegate(std::make_shared<FrameWriterInterfaceHelper>([this](const Frame::Ptr &frame) {
     /* Chỗ này được gọi vào MediaSink::inputFrame 
     * Tại đây tiếp tục tìm trong _track_map để lấy ra các track đã có trong map để send frame xuống
     * auto ret = it->second.first->inputFrame(frame) chính là FrameDispatcher.inputFrame;
     */   
          return _sink->inputFrame(frame);
     }));
     return true;
}

#### Giải thích _track_map ở trên #######
Trước đó trong hàm setDirectProxy(); có add các track của video và audio
--- _muxer->addTrack(videoTrack);
--- _muxer->addTrack(audioTrack);
addTrack này được gọi trong MediaSink::addTrack(...) đã add các track vào _track_map có một addDelegate
track->addDelegate(std::make_shared<FrameWriterInterfaceHelper>([this](const Frame::Ptr &frame) {
     if (_all_track_ready) {
          return onTrackFrame(frame);
     }
     ....
}));
addDelegate này tạo một smart-pointer là FrameWriterInterfaceHelper
Trong FrameWriterInterfaceHelper có một hàm inputFrame được khai báo như sau:
bool inputFrame(const Frame::Ptr &frame) override { return _writeCallback(frame); }
Nghĩa là khi gọi vào inputFrame thì nó sẽ return ra một Callback được thả vào, do đó nó sẽ gọi xuống 
if (_all_track_ready) {
     return onTrackFrame(frame);
}
onTrackFrame ở đây chính là MultiMediaSourceMuxer::onTrackFrame
tại đây gọi vào hls->inputFrame(frame);
hls->inputFrame lại là hàm được kế thừa từ MpegMuxer trong file MGEG.cpp
Tại MpegMuxer::inputFrame sẽ thực hiện việc giải mã video sử dụng mpeg-muxer.h
Sau đó gọi tới:
--- MpegMuxer::flushCache()
--- onWrite(std::move(_current_buffer), _timestamp, _key_pos);
onWrite là một hàm trừu tượng của lớp MpegMuxer nên nó được thực thi tại:
HlsRecorder.h vì được kế thừa MpegMuxer.
void onWrite(std::shared_ptr<toolkit::Buffer> buffer, uint64_t timestamp, bool key_pos) override {
     if (!buffer) {
          _hls->inputData(nullptr, 0, timestamp, key_pos);
     } else {
          /* Trong hàm này bắt đầu ghi xuống file m3u8 với EXT-X-PLAYLIST-TYPE:EVENT 
          * TYPE:EVENT nghĩa là nói với trình duyệt đây là file live các segment sẽ tiếp tục được đẩy vào.
          */
          _hls->inputData(buffer->data(), buffer->size(), timestamp, key_pos);
     }
}
```

<strong>Trong hàm setDirectProxy(); </strong>

mediaSource = std::make_shared<RtmpMediaSource> hoặc RtspMediaSource

setMediaSource(mediaSource);

<strong>Trong PlayerProxy.cpp.</strong>

Dòng 60: strongSelf->onPlaySuccess();
     
Tạo được một _muxer = std::make_shared<MultiMediaSourceMuxer>.
``` c++
add thêm các track để tách các luồng video với nhau.
_muxer->addTrack(videoTrack);
_muxer->addTrack(audioTrack);

if (_media_src) {
     /* Set mediaSource Listener _muxer */
     _media_src->setListener(_muxer);
}
```

<strong>Trong server/WebApi.cpp.</strong>

api_regist("/index/api/startRecord");
``` c++
auto src = MediaSource::find(allArgs["vhost"], allArgs["app"], allArgs["stream"] );
... 
auto result = src->setupRecord(...)

Dòng này gọi đến: return listener->setupRecord(*this, type, start, custom_path, max_second);
listener được set là MultiMediaSourceMuxer ở phía trên --> MultiMediaSourceMuxer.setupRecord()
Tại MultiMediaSourceMuxer::setupRecord sẽ makeRecorder sẽ gọi đến HlsRecorder.
HlsRecorder đồng thời gọi luôn lớp cha là MpegMuxer để khởi tạo một:
_context = (struct mpeg_muxer_t *) mpeg_muxer_create(_is_ps, &func, this);
mpeg-muxer.h dùng để giải mã luồng video, audio cho việc ghi xuống file .ts

```

Như vậy là từ các máy quay của trận bóng đá sẽ tạo được ra các file m3u8 rồi stream đến nhiều nền tảng khác nhau dựa trên giao thức http thay vì rtmp (tại rtmp hay rtsp phải có software client mới mở được)

HLS thường có độ trể từ 5- 20 giây so với máy quay là vì chúng ta cứ 5s sẽ ghi xuống file m3u8 một lần. Nhưng ghi được ở nhiều độ phân giải khác nhau nên client tuỳ vào tốc độ mạng sẽ xem được các độ phân giải khác nhau tránh được dán đoạn chương trình khi mạng yếu hơn.
<p>Sau khi ghi xong file ts thì excute lệnh ffmpeg để resize ra nhiều file có độ phân giải khác nhau.</p>

## File m3u8 cho nhiều độ phân giải khác nhau
``` text
#EXTM3U
#EXT-X-STREAM-INF:PROGRAM-ID=1,BANDWIDTH=1280000
http://example.com/low.m3u8
#EXT-X-STREAM-INF:PROGRAM-ID=1,BANDWIDTH=2560000
http://example.com/mid.m3u8
#EXT-X-STREAM-INF:PROGRAM-ID=1,BANDWIDTH=7680000
http://example.com/hi.m3u8
```

## RTMP PUBLIC AND PLAY.
```text
static onceToken token([]() {
     ...
     s_cmd_functions.emplace("publish", &RtmpSession::onCmd_publish);
     s_cmd_functions.emplace("play", &RtmpSession::onCmd_play);
     ...
});

VD: Có Có 10 RtmpSession thì (1 public, 9 player).
Người phát sẽ thực hiện lệnh publish,
Người play sẽ thực hiện lệnh play.

Session người public sẽ tạo được 
_push_src = std::make_shared<RtmpMediaSourceImp>(_media_info._vhost, _media_info._app, _media_info._streamid);

Session của người play sẽ tìm cái source này để play.
     _ring_reader = src->getRing()->attach(getPoller());
     _ring_reader->setReadCB(...): Set một calback cho đọc các frame từ người phát.

Trong session của người phát có RtmpSession::onRtmpChunk(...) sẽ thực hiện gửi các frame xuống 
     _push_src->onWrite(std::move(packet));
     ....
     _ring->write(std::move(rtmp_list), _have_video ? key_pos : true); gọi vào 
     _RingReaderDispatcher::write --> reader->onRead(in, is_key);
     reader->onRead chính là callback đã set ở trên.
     Trong callback _ring_reader->setReadCB(...) sẽ gửi các frame xuống session play.
```

## build & run
``` sh
Chi tiết xem trong file README_en.md trước khi build.
cmake .
mkdir build
cp -r server/* build/
cd build
make
make install
Chạy bằng lệnh: /usr/local/bin/MediaServer
Hoặc tạo file sh để chạy backgroud cho file MediaServer.
```
