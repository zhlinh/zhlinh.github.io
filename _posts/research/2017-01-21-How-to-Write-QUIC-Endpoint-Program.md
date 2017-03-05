---
layout: article
title:  "How to Write a QUIC Endpoint Program"
categories: research
tags: [QUIC, Google, Codes]
toc: true
image:
    teaser: research/2017-01-21-How-to-Write-QUIC-Endpoint-Program/teaser.jpg
date: 2017-01-21
---

要编写使用QUIC的应用程序，需要创建一些新类以及继承一些QUIC类。 本文档通过QUIC的
客户端和服务器程序作为示例，介绍如何在自己的应用程序中使用QUIC。

使用的proto-quic版本为：[Updating to 56.0.2912.0 \(#15\)](https://github.com/google/proto-quic/commit/0db5f234b86699c182394bb261ee226ab800b60f)。

---

## 1 新类

为了方便处理所有的QUIC类，可能需要创建封装客户端和服务器功能的类。 此外，至少需
要创建实现QuicPacketWriter接口的类。

### 1.1 QUIC客户端

需要一个类，以创建一个QuicSimpleClient类\[src/net/tools/quic_simple_client.h&.cc\]
( 其基类是QuicClientBase类\[src/net/tools/quic_client_base.h&.cc\]
和 QuicChromiumPacketReader::Visitor类\[src/net/quic/chromium/quic_chromium_packet_reader.h&cc\]，
QuicClientBase的基类是QuicClientPushPromiseIndex::Delegate和QuicSpdyStream::Visitor)，
连接类和数据包写入器，并连接所有这些类。 它还必须读取网络数据包并将它们分派到会话中。

QUIC客户端的二进制文件为quic_client\[src/net/tools/quic/quic_simple_client_bin.cc\]，
在main函数中调用QuicSimpleClient类的基本功能方法。其流程如下图所示。

![Pic-1.1-Client-Implementation](/images/research/2017-01-21-How-to-Write-QUIC-Endpoint-Program/Pic-1.1-Client-Implementation.jpg)
<center>图1.1 Client Implementation</center>

以下对每一步进行详细介绍：

创建QUIC客户端可以分为下面的几个步骤。 首先，必须确定要连接的IP地址，服务器ID和
要使用的QUIC版本。 服务器ID作为所要连接到的服务器的标识，是由HTTP方案（http或
https），主机和端口组成的。 示例客户端接收构造函数中的这些参数（还包括epoll服务
器）并将它们存储为属性。

~~~c++
QuicSimpleClient::QuicSimpleClient(
    IPEndPoint server_address,
    const QuicServerId& server_id,
    const QuicVersionVector& supported_versions,
    std::unique_ptr<ProofVerifier> proof_verifier)
    : QuicSimpleClient(server_address,
                       server_id,
                       supported_versions,
                       QuicConfig(),
                       std::move(proof_verifier)) {}

QuicSimpleClient::QuicSimpleClient(
    IPEndPoint server_address,
    const QuicServerId& server_id,
    const QuicVersionVector& supported_versions,
    const QuicConfig& config,
    std::unique_ptr<ProofVerifier> proof_verifier)
    : QuicClientBase(server_id,
                     supported_versions,
                     config,
                     CreateQuicConnectionHelper(),
                     CreateQuicAlarmFactory(),
                     std::move(proof_verifier)),
      initialized_(false),
      packet_reader_started_(false),
      weak_factory_(this) {
  set_server_address(server_address);
}
~~~

然后初始化客户端以创建连接，应该在调用Connect函数之前。打开UDP套接字，设置接收和
发送缓冲区大小，并在添加QuicChromiumPacketReader类
\[src/net/quic/chromium/quic_chromium_packet_reader.h&cc\]。

~~~c++
bool QuicClientBase::Initialize() {

  ...

  // If an initial flow control window has not explicitly been set, then use the
  // same values that Chrome uses.
  const uint32_t kSessionMaxRecvWindowSize = 15 * 1024 * 1024;  // 15 MB
  const uint32_t kStreamMaxRecvWindowSize = 6 * 1024 * 1024;    //  6 MB

  ...

  if (!CreateUDPSocketAndBind(server_address_, bind_to_address_, local_port_)) {
    return false;
  }

  initialized_ = true;
  return true;
}


bool QuicSimpleClient::CreateUDPSocketAndBind(IPEndPoint server_address,
                                              IPAddress bind_to_address,
                                              int bind_to_port) {
  std::unique_ptr<UDPClientSocket> socket(
      new UDPClientSocket(DatagramSocket::DEFAULT_BIND, RandIntCallback(),
                          &net_log_, NetLogSource()));

  int address_family = server_address.GetSockAddrFamily();
  if (bind_to_address.size() != 0) {
    client_address_ = IPEndPoint(bind_to_address, bind_to_port);
  } else if (address_family == AF_INET) {
    client_address_ = IPEndPoint(IPAddress::IPv4AllZeros(), bind_to_port);
  } else {
    client_address_ = IPEndPoint(IPAddress::IPv6AllZeros(), bind_to_port);
  }

  int rc = socket->Connect(server_address);
  if (rc != OK) {
    LOG(ERROR) << "Connect failed: " << ErrorToShortString(rc);
    return false;
  }

  rc = socket->SetReceiveBufferSize(kDefaultSocketReceiveBuffer);
  if (rc != OK) {
    LOG(ERROR) << "SetReceiveBufferSize() failed: " << ErrorToShortString(rc);
    return false;
  }

  rc = socket->SetSendBufferSize(kDefaultSocketReceiveBuffer);
  if (rc != OK) {
    LOG(ERROR) << "SetSendBufferSize() failed: " << ErrorToShortString(rc);
    return false;
  }

  rc = socket->GetLocalAddress(&client_address_);
  if (rc != OK) {
    LOG(ERROR) << "GetLocalAddress failed: " << ErrorToShortString(rc);
    return false;
  }

  socket_.swap(socket);

  // std::unique_ptr<QuicChromiumPacketReader> packet_reader_
  packet_reader_.reset(new QuicChromiumPacketReader(
      socket_.get(), &clock_, this, kQuicYieldAfterPacketsRead,
      QuicTime::Delta::FromMilliseconds(kQuicYieldAfterDurationMilliseconds),
      NetLogWithSource()));

  if (socket != nullptr) {
    socket->Close();
  }

  return true;
}


class NET_EXPORT_PRIVATE QuicChromiumPacketReader {
 public:
  class NET_EXPORT_PRIVATE Visitor {
   public:
    virtual ~Visitor() {}
    virtual void OnReadError(int result,
                             const DatagramClientSocket* socket) = 0;
    virtual bool OnPacket(const QuicReceivedPacket& packet,
                          IPEndPoint local_address,
                          IPEndPoint peer_address) = 0;
  };

  QuicChromiumPacketReader(DatagramClientSocket* socket,
                           QuicClock* clock,
                           Visitor* visitor,
                           int yield_after_packets,
                           QuicTime::Delta yield_after_duration,
                           const NetLogWithSource& net_log);
 ...
}
~~~

在UDP套接字打开的情况下，可以打开QUIC连接，连接到QUIC服务器，包括执行同步加密握手。
首先创建一个包写入器QuickPacketWriter并初始化一个新的QuicConnection类
\[src/net/quic/core/quic_connection.h&cc\]实例。 然后我们可以使用该连接创建一个
新的QuicClientSession\[src/net/quic/core/quic_client_session.h&cc\]
( 其基类是QuicClientSessionBase类， 而QuicClientSessionBase类的基类是
QuicSpdySession类和QuicCryptoClientStream::ProofHandler类，而QuicSpdySession类
的基类是QuicSession类，而QuicSession类实现QuicConnectionVisitorInterface接口)实例。 接着初始化会话，最后我们可以启动加密握手并等待它完成。

~~~c++
bool QuicClientBase::Connect() {
  // Attempt multiple connects until the maximum number of client hellos have
  // been sent.
  while (!connected() &&
         GetNumSentClientHellos() <= QuicCryptoClientStream::kMaxClientHellos) {
    StartConnect();
    while (EncryptionBeingEstablished()) {
      WaitForEvents();
    }
    if (FLAGS_enable_quic_stateless_reject_support && connected()) {
      // Resend any previously queued data.
      ResendSavedData();
    }
    if (session() != nullptr &&
        session()->error() != QUIC_CRYPTO_HANDSHAKE_STATELESS_REJECT) {
      // We've successfully created a session but we're not connected, and there
      // is no stateless reject to recover from.  Give up trying.
      break;
    }
  }
  if (!connected() &&
      GetNumSentClientHellos() > QuicCryptoClientStream::kMaxClientHellos &&
      session() != nullptr &&
      session()->error() == QUIC_CRYPTO_HANDSHAKE_STATELESS_REJECT) {
    // The overall connection failed due too many stateless rejects.
    set_connection_error(QUIC_CRYPTO_TOO_MANY_REJECTS);
  }
  return session()->connection()->connected();
}


void QuicClientBase::StartConnect() {
  DCHECK(initialized_);
  DCHECK(!connected());

  QuicPacketWriter* writer = CreateQuicPacketWriter();

  if (connected_or_attempting_connect()) {
    // If the last error was not a stateless reject, then the queued up data
    // does not need to be resent.
    if (session()->error() != QUIC_CRYPTO_HANDSHAKE_STATELESS_REJECT) {
      ClearDataToResend();
    }
    // Before we destroy the last session and create a new one, gather its stats
    // and update the stats for the overall connection.
    UpdateStats();
  }

  CreateQuicClientSession(new QuicConnection(
      GetNextConnectionId(), server_address(), helper(), alarm_factory(),
      writer,
      /* owns_writer= */ false, Perspective::IS_CLIENT, supported_versions()));

  // Reset |writer()| after |session()| so that the old writer outlives the old
  // session.
  set_writer(writer);
  // QuicClientSession* session() { return session_.get(); }
  session()->Initialize();
  session()->CryptoConnect();
  set_connected_or_attempting_connect(true);
}

QuicPacketWriter* QuicSimpleClient::CreateQuicPacketWriter() {
  return new QuicChromiumPacketWriter(socket_.get());
}

QuicClientSession* QuicClientBase::CreateQuicClientSession(
    QuicConnection* connection) {
  session_.reset(new QuicClientSession(config_, connection, server_id_,
                                       &crypto_config_, &push_promise_index_));
  if (initial_max_packet_length_ != 0) {
    session()->connection()->SetMaxPacketLength(initial_max_packet_length_);
  }
  return session_.get();
}
~~~


要发送数据，客户端调用SendRequest方法，需要创建
QuicSpdyClientStream类\[src/net/tools/quic/quic_spdy_client_stream.h&cc\]
( 其基类是QuicSpdyStream类，而QuicSpdyStream类的基类是ReliableQuicStream类 )
实例。 新的QUIC流QuicSpdyClientStream实例必须由CreateReliableClientStream方法
调用会话的CreateOutgoingDynamicStream方法创建，因为新的流( stream )必须调用会
话的基类( QuicSession类 )中受保护( protected )的ActivateStream方法。

在添加并激活信的流后，客户端的SendRequest方法最后会调用QuicSpdyClientStream类
的SendRequest方法会调用流基类ReliableQuicStream类中的WriteOrBufferData方法将
数据写入流( stream )。

~~~c++
void QuicClientBase::SendRequest(const SpdyHeaderBlock& headers,
                                 StringPiece body,
                                 bool fin) {
  QuicClientPushPromiseIndex::TryHandle* handle;
  QuicAsyncStatus rv = push_promise_index()->Try(headers, this, &handle);
  if (rv == QUIC_SUCCESS)
    return;

  if (rv == QUIC_PENDING) {
    // May need to retry request if asynchronous rendezvous fails.
    AddPromiseDataToResend(headers, body, fin);
    return;
  }

  QuicSpdyClientStream* stream = CreateReliableClientStream();
  if (stream == nullptr) {
    QUIC_BUG << "stream creation failed!";
    return;
  }
  stream->SendRequest(headers.Clone(), body, fin);
  // Record this in case we need to resend.
  MaybeAddDataToResend(headers, body, fin);
}


QuicSpdyClientStream* QuicClientBase::CreateReliableClientStream() {
  if (!connected()) {
    return nullptr;
  }

  QuicSpdyClientStream* stream =
      session_->CreateOutgoingDynamicStream(kDefaultPriority);
  if (stream) {

    // set QuicSimpleClient as the vistor
    stream->set_visitor(this);
  }
  return stream;
}


QuicSpdyClientStream* QuicClientSession::CreateOutgoingDynamicStream(
    SpdyPriority priority) {
  if (!ShouldCreateOutgoingDynamicStream()) {
    return nullptr;
  }
  std::unique_ptr<QuicSpdyClientStream> stream = CreateClientStream();
  stream->SetPriority(priority);
  QuicSpdyClientStream* stream_ptr = stream.get();
  ActivateStream(std::move(stream));
  return stream_ptr;
}

std::unique_ptr<QuicSpdyClientStream> QuicClientSession::CreateClientStream() {
  return base::MakeUnique<QuicSpdyClientStream>(GetNextOutgoingStreamId(),
                                                this);
}


void QuicSession::ActivateStream(std::unique_ptr<ReliableQuicStream> stream) {
  QuicStreamId stream_id = stream->id();
  DVLOG(1) << ENDPOINT << "num_streams: " << dynamic_stream_map_.size()
           << ". activating " << stream_id;
  DCHECK(!base::ContainsKey(dynamic_stream_map_, stream_id));
  DCHECK(!base::ContainsKey(static_stream_map_, stream_id));
  dynamic_stream_map_[stream_id] = std::move(stream);
  if (IsIncomingStream(stream_id)) {
    ++num_dynamic_incoming_streams_;
  }
  // Increase the number of streams being emulated when a new one is opened.
  connection_->SetNumOpenStreams(dynamic_stream_map_.size());
}


size_t QuicSpdyClientStream::SendRequest(SpdyHeaderBlock headers,
                                         StringPiece body,
                                         bool fin) {
  bool send_fin_with_headers = fin && body.empty();
  size_t bytes_sent = body.size();
  header_bytes_written_ =
      WriteHeaders(std::move(headers), send_fin_with_headers, nullptr);
  bytes_sent += header_bytes_written_;

  if (!body.empty()) {
    WriteOrBufferData(body, fin, nullptr);
  }

  return bytes_sent;
}
~~~

接着客户端等待接收数据，调用WaitForResponse方法读取Socket数据并进行处理。

~~~c++
bool QuicClientBase::WaitForEvents() {
  DCHECK(connected());

  RunEventLoop();

  DCHECK(session() != nullptr);
  if (!connected() &&
      session()->error() == QUIC_CRYPTO_HANDSHAKE_STATELESS_REJECT) {
    DCHECK(FLAGS_enable_quic_stateless_reject_support);
    DVLOG(1) << "Detected stateless reject while waiting for events.  "
             << "Attempting to reconnect.";
    Connect();
  }

  return session()->num_active_requests() != 0;
}


void QuicSimpleClient::RunEventLoop() {
  StartPacketReaderIfNotStarted();
  base::RunLoop().RunUntilIdle();
}

void QuicSimpleClient::StartPacketReaderIfNotStarted() {
  if (!packet_reader_started_) {
    packet_reader_->StartReading();
    packet_reader_started_ = true;
  }
}

void QuicChromiumPacketReader::StartReading() {
  if (read_pending_)
    return;

  if (num_packets_read_ == 0)
    yield_after_ = clock_->Now() + yield_after_duration_;

  DCHECK(socket_);
  read_pending_ = true;
  int rv = socket_->Read(read_buffer_.get(), read_buffer_->size(),
                         base::Bind(&QuicChromiumPacketReader::OnReadComplete,
                                    weak_factory_.GetWeakPtr()));
  UMA_HISTOGRAM_BOOLEAN("Net.QuicSession.AsyncRead", rv == ERR_IO_PENDING);
  if (rv == ERR_IO_PENDING) {
    num_packets_read_ = 0;
    return;
  }

  if (++num_packets_read_ > yield_after_packets_ ||
      clock_->Now() > yield_after_) {
    num_packets_read_ = 0;
    // Data was read, process it.
    // Schedule the work through the message loop to 1) prevent infinite
    // recursion and 2) avoid blocking the thread for too long.
    base::ThreadTaskRunnerHandle::Get()->PostTask(
        FROM_HERE, base::Bind(&QuicChromiumPacketReader::OnReadComplete,
                              weak_factory_.GetWeakPtr(), rv));
  } else {
    OnReadComplete(rv);
  }


void QuicChromiumPacketReader::OnReadComplete(int result) {
  read_pending_ = false;
  if (result == 0)
    result = ERR_CONNECTION_CLOSED;

  if (result < 0) {
    visitor_->OnReadError(result, socket_);
    return;
  }

  QuicReceivedPacket packet(read_buffer_->data(), result, clock_->Now());
  IPEndPoint local_address;
  IPEndPoint peer_address;
  socket_->GetLocalAddress(&local_address);
  socket_->GetPeerAddress(&peer_address);
  if (!visitor_->OnPacket(packet, local_address, peer_address))
    return;

  StartReading();
}
~~~

此外，客户端还必须读取通知的UDP数据包。QuicChromiumPacketReader::OnReadComplete
方法会注册为Socket的回调函数，继而调用客户端的OnPacket方法。所以客户端实现了
QuicChromiumPacketReader::Visitor接口的OnPacket方法。

当成功读取QUIC包时，必须使用ProcessUdpPacket方法将其分派到QuicConnection中，最终
会交给QuicFramer类的ProcessPacket方法处理。

~~~c++
bool QuicSimpleClient::OnPacket(const QuicReceivedPacket& packet,
                                IPEndPoint local_address,
                                IPEndPoint peer_address) {
  session()->connection()->ProcessUdpPacket(local_address, peer_address,
                                            packet);
  if (!session()->connection()->connected()) {
    return false;
  }

  return true;
}


void QuicConnection::ProcessUdpPacket(const IPEndPoint& self_address,
                                      const IPEndPoint& peer_address,
                                      const QuicReceivedPacket& packet) {
  if (!connected_) {
    return;
  }
  if (debug_visitor_ != nullptr) {
    debug_visitor_->OnPacketReceived(self_address, peer_address, packet);
  }
  last_size_ = packet.length();
  current_packet_data_ = packet.data();

  last_packet_destination_address_ = self_address;
  last_packet_source_address_ = peer_address;
  if (!IsInitializedIPEndPoint(self_address_)) {
    self_address_ = last_packet_destination_address_;
  }
  if (!IsInitializedIPEndPoint(peer_address_)) {
    peer_address_ = last_packet_source_address_;
  }

  stats_.bytes_received += packet.length();
  ++stats_.packets_received;

  // Ensure the time coming from the packet reader is within a minute of now.
  if (FLAGS_quic_allow_large_send_deltas &&
      std::abs((packet.receipt_time() - clock_->ApproximateNow()).ToSeconds()) >
          60) {
    QUIC_BUG << "Packet receipt time:"
             << packet.receipt_time().ToDebuggingValue()
             << " too far from current time:"
             << clock_->ApproximateNow().ToDebuggingValue();
  }
  time_of_last_received_packet_ = packet.receipt_time();
  DVLOG(1) << ENDPOINT << "time of last received packet: "
           << time_of_last_received_packet_.ToDebuggingValue();

  ScopedRetransmissionScheduler alarm_delayer(this);
  if (!framer_.ProcessPacket(packet)) {
    // If we are unable to decrypt this packet, it might be
    // because the CHLO or SHLO packet was lost.
    if (framer_.error() == QUIC_DECRYPTION_FAILURE) {
      if (encryption_level_ != ENCRYPTION_FORWARD_SECURE &&
          undecryptable_packets_.size() < max_undecryptable_packets_) {
        QueueUndecryptablePacket(packet);
      } else if (debug_visitor_ != nullptr) {
        debug_visitor_->OnUndecryptablePacket();
      }
    }
    DVLOG(1) << ENDPOINT << "Unable to process packet.  Last packet processed: "
             << last_header_.packet_number;
    current_packet_data_ = nullptr;
    return;
  }

  ++stats_.packets_processed;
  if (active_peer_migration_type_ != NO_CHANGE &&
      sent_packet_manager_->GetLargestObserved(last_header_.path_id) >
          highest_packet_sent_before_peer_migration_) {
    OnPeerMigrationValidated(last_header_.path_id);
  }
  MaybeProcessUndecryptablePackets();
  MaybeSendInResponseToPacket();
  SetPingAlarm();
  current_packet_data_ = nullptr;
}
~~~

此外，当Stream收到FIN标志为1的帧后会调用Stream->OnClose()方法，继而会调用
QuicSimpleClient(即QuicClientBase)的OnClose方法，取出Stream中收到的HTTP响应头部
和数据。所以客户端实现了QuicSpdyStream::Visitor接口中的OnClose方法。

~~~c++
void QuicClientBase::OnClose(QuicSpdyStream* stream) {
  DCHECK(stream != nullptr);
  QuicSpdyClientStream* client_stream =
      static_cast<QuicSpdyClientStream*>(stream);

  const SpdyHeaderBlock& response_headers = client_stream->response_headers();
  if (response_listener_ != nullptr) {
    response_listener_->OnCompleteResponse(stream->id(), response_headers,
                                           client_stream->data());
  }

  // Store response headers and body.
  if (store_response_) {
    auto status = response_headers.find(":status");
    if (status == response_headers.end() ||
        !StringToInt(status->second, &latest_response_code_)) {
      LOG(ERROR) << "Invalid response headers";
    }
    latest_response_headers_ = response_headers.DebugString();
    latest_response_header_block_ = response_headers.Clone();
    latest_response_body_ = client_stream->data();
    latest_response_trailers_ =
        client_stream->received_trailers().DebugString();
  }
}
~~~

### 1.2 QUIC服务器

服务器端更复杂，因为它必须处理多个连接的多个会话。 因此，服务器QuicSimpleServer类
\[src/net/tools/quic_simple_server.h&.cc\]通常使用分派器类（QuicDispatcher）。
它将处理类似于客户端（使用epoll）的网络分组的读取，但不是直接将其交给连接，
而是将其传递给分派器。

QUIC服务器的二进制文件为quic_server\[src/net/tools/quic/quic_simple_server_bin.cc\]，
在main函数中调用QuicSimpleServer类的基本功能方法。其流程如下图所示。

![Pic-1.2-Server-Implementation](/images/research/2017-01-21-How-to-Write-QUIC-Endpoint-Program/Pic-1.2-Server-Implementation.jpg)
<center>图1.2 Server Implementation</center>

以下对每一步进行详细介绍：

首先，服务器QuicSimpleServer类的实例需要一个QuicConfig( 稍后会传递给会话 )，
一个QuicCryptoServerConfig::ConfigOptions( 使用默认配置 ) 和一个QuicVersionVector
( 存储所支持的QUIC版本的数组 )来初始化。

~~~c++
QuicSimpleServer::QuicSimpleServer(
    std::unique_ptr<ProofSource> proof_source,
    const QuicConfig& config,
    const QuicCryptoServerConfig::ConfigOptions& crypto_config_options,
    const QuicVersionVector& supported_versions)
    : version_manager_(supported_versions),
      helper_(
          new QuicChromiumConnectionHelper(&clock_, QuicRandom::GetInstance())),
      alarm_factory_(new QuicChromiumAlarmFactory(
          base::ThreadTaskRunnerHandle::Get().get(),
          &clock_)),
      config_(config),
      crypto_config_options_(crypto_config_options),
      crypto_config_(kSourceAddressTokenSecret,
                     QuicRandom::GetInstance(),
                     std::move(proof_source)),
      read_pending_(false),
      synchronous_read_count_(0),
      read_buffer_(new IOBufferWithSize(kReadBufferSize)),
      weak_factory_(this) {
  Initialize();
}


void QuicSimpleServer::Initialize() {

  ...

  // If an initial flow control window has not explicitly been set, then use a
  // sensible value for a server: 1 MB for session, 64 KB for each stream.
  const uint32_t kInitialSessionFlowControlWindow = 1 * 1024 * 1024;  // 1 MB
  const uint32_t kInitialStreamFlowControlWindow = 64 * 1024;         // 64 KB

  ...

  std::unique_ptr<CryptoHandshakeMessage> scfg(crypto_config_.AddDefaultConfig(
      helper_->GetRandomGenerator(), helper_->GetClock(),
      crypto_config_options_));
}
~~~

然后开始在UDP套接字上侦听并绑定到请求的地址。 使用该套接字初始化一个新的分派器。

~~~c++
int QuicSimpleServer::Listen(const IPEndPoint& address) {
  std::unique_ptr<UDPServerSocket> socket(
      new UDPServerSocket(&net_log_, NetLogSource()));

  socket->AllowAddressReuse();

  int rc = socket->Listen(address);
  ...

  // These send and receive buffer sizes are sized for a single connection,
  // because the default usage of QuicSimpleServer is as a test server with
  // one or two clients.  Adjust higher for use with many clients.
  rc = socket->SetReceiveBufferSize(
      static_cast<int32_t>(kDefaultSocketReceiveBuffer));
  ...

  rc = socket->SetSendBufferSize(20 * kMaxPacketSize);
  ...

  rc = socket->GetLocalAddress(&server_address_);
  ...

  socket_.swap(socket);

  dispatcher_.reset(new QuicSimpleDispatcher(
      config_, &crypto_config_, &version_manager_,
      std::unique_ptr<QuicConnectionHelperInterface>(helper_),
      std::unique_ptr<QuicCryptoServerStream::Helper>(
          new QuicSimpleServerSessionHelper(QuicRandom::GetInstance())),
      std::unique_ptr<QuicAlarmFactory>(alarm_factory_)));
  QuicSimpleServerPacketWriter* writer =
      new QuicSimpleServerPacketWriter(socket_.get(), dispatcher_.get());
  dispatcher_->InitializeWithWriter(writer);

  StartReading();

  return OK;
}
~~~

服务器以多线程( 用以承载多个会话 )准备读或写套接字。 随后可以阻塞读取UDP分组，
如果读取了32次或无分组到达则调用OnReadComplete方法，并由此创建QUIC分组
QuicReceivedPacket传递给分派器QuicSimpleDispatcher类
\[src/net/tools/quic/quic_simple_dispatcher.h&cc\] ( 其基类是QuicDispatcher类
\[src/net/tools/quic/quic_dispatcher.h&cc\]，QuicDispatcher类的基类是
QuicTimeWaitListManager::Visitor类\[src/net/tools/quic/quic_time_wait_list_manager.h&cc\]、
ProcessPacketInterface类\[src/net/tools/quic/quic_process_packet_interface.h\]、
QuicBlockedWriterInterface类、QuicFramerVisitorInterface类和
QuicBufferedPacketStore::VisitorInterface类 ，QuicTimeWaitListManager::Visitor类
的基类是QuicSession::Visitor)的ProcessPacket方法处理。

~~~c++
void QuicSimpleServer::StartReading() {
  if (synchronous_read_count_ == 0) {
    // Only process buffered packets once per message loop.
    // set the max connections to create
    dispatcher_->ProcessBufferedChlos(kNumSessionsToCreatePerSocketEvent);
  }

  if (read_pending_) {
    return;
  }
  read_pending_ = true;

  int result = socket_->RecvFrom(
      read_buffer_.get(), read_buffer_->size(), &client_address_,
      base::Bind(&QuicSimpleServer::OnReadComplete, base::Unretained(this)));

  if (result == ERR_IO_PENDING) {
    synchronous_read_count_ = 0;
    if (dispatcher_->HasChlosBuffered()) {
      // No more packets to read, so yield before processing buffered packets.
      base::ThreadTaskRunnerHandle::Get()->PostTask(
          FROM_HERE, base::Bind(&QuicSimpleServer::StartReading,
                                weak_factory_.GetWeakPtr()));
    }
    return;
  }

  if (++synchronous_read_count_ > 32) {
    synchronous_read_count_ = 0;
    // Schedule the processing through the message loop to 1) prevent infinite
    // recursion and 2) avoid blocking the thread for too long.
    base::ThreadTaskRunnerHandle::Get()->PostTask(
        FROM_HERE, base::Bind(&QuicSimpleServer::OnReadComplete,
                              weak_factory_.GetWeakPtr(), result));
  } else {
    OnReadComplete(result);
  }
}


void QuicSimpleServer::OnReadComplete(int result) {
  read_pending_ = false;
  if (result == 0)
    result = ERR_CONNECTION_CLOSED;

  if (result < 0) {
    LOG(ERROR) << "QuicSimpleServer read failed: " << ErrorToString(result);
    Shutdown();
    return;
  }

  QuicReceivedPacket packet(read_buffer_->data(), result,
                            helper_->GetClock()->Now(), false);
  dispatcher_->ProcessPacket(server_address_, client_address_, packet);

  StartReading();
}
~~~

分派器使用成帧器QuicFramer类\[src/net/quic/core/quic_framer.h&cc\]来解析QUIC数据
包，会在读取帧时调用分派器(实现了QuicFramerVisitorInterface接口)的OnPacket、
OnUnauthenticatedPublicHeader等方法。

~~~c++
void QuicDispatcher::ProcessPacket(const IPEndPoint& server_address,
                                   const IPEndPoint& client_address,
                                   const QuicReceivedPacket& packet) {
  current_server_address_ = server_address;
  current_client_address_ = client_address;
  current_packet_ = &packet;
  // ProcessPacket will cause the packet to be dispatched in
  // OnUnauthenticatedPublicHeader, or sent to the time wait list manager
  // in OnUnauthenticatedHeader.
  framer_.ProcessPacket(packet);
  // TODO(rjshade): Return a status describing if/why a packet was dropped,
  //                and log somehow.  Maybe expose as a varz.
}


bool QuicFramer::ProcessPacket(const QuicEncryptedPacket& packet) {
  QuicDataReader reader(packet.data(), packet.length());

  visitor_->OnPacket();

  // First parse the public header.
  QuicPacketPublicHeader public_header;
  if (!ProcessPublicHeader(&reader, &public_header)) {
    DVLOG(1) << "Unable to process public header.";
    DCHECK_NE("", detailed_error_);
    return RaiseError(QUIC_INVALID_PACKET_HEADER);
  }

  if (!visitor_->OnUnauthenticatedPublicHeader(public_header)) {
    // The visitor suppresses further processing of the packet.
    return true;
  }

  if (perspective_ == Perspective::IS_SERVER && public_header.version_flag &&
      public_header.versions[0] != quic_version_) {
    if (!visitor_->OnProtocolVersionMismatch(public_header.versions[0])) {
      return true;
    }
  }

  bool rv;
  if (perspective_ == Perspective::IS_CLIENT && public_header.version_flag) {
    rv = ProcessVersionNegotiationPacket(&reader, &public_header);
  } else if (public_header.reset_flag) {
    rv = ProcessPublicResetPacket(&reader, public_header);
  } else if (packet.length() <= kMaxPacketSize) {
    // The optimized decryption algorithm implementations run faster when
    // operating on aligned memory.
    //
    // TODO(rtenneti): Change the default 64 alignas value (used the default
    // value from CACHELINE_SIZE).
    ALIGNAS(64) char buffer[kMaxPacketSize];
    rv = ProcessDataPacket(&reader, public_header, packet, buffer,
                           kMaxPacketSize);
  } else {
    std::unique_ptr<char[]> large_buffer(new char[packet.length()]);
    rv = ProcessDataPacket(&reader, public_header, packet, large_buffer.get(),
                           packet.length());
    QUIC_BUG_IF(rv) << "QUIC should never successfully process packets larger"
                    << "than kMaxPacketSize. packet size:" << packet.length();
  }

  return rv;
}
~~~

分派器的OnUnauthenticatedPublicHeader方法通过公共头部中的连接ID在会话字典中查找
相应会话。 当分派器确定QUIC包所属的连接时，它将调用会话QuicSimpleServerSession类
\[src/net/tools/quic/quic_simple_server_session.h&cc\] (其基类是
QuicServerSessionBase类，而QuicServerSessionBase类的基类是QuicSpdySession类，
而QuicSpdySession类的基类是QuicSession类，而QuicSession类实现
QuicConnectionVisitorInterface接口 )的QuicSession::ProcessUdpPacket方法分派到
QuicConnection中。 如果没有这样的会话，分派器将会先创建它。

~~~c++
bool QuicDispatcher::OnUnauthenticatedPublicHeader(
    const QuicPacketPublicHeader& header) {
  current_connection_id_ = header.connection_id;

  // Port zero is only allowed for unidirectional UDP, so is disallowed by QUIC.
  // Given that we can't even send a reply rejecting the packet, just drop the
  // packet.
  if (current_client_address_.port() == 0) {
    return false;
  }

  // Stopgap test: The code does not construct full-length connection IDs
  // correctly from truncated connection ID fields.  Prevent this from causing
  // the connection ID lookup to error by dropping any packet with a short
  // connection ID.
  if (header.connection_id_length != PACKET_8BYTE_CONNECTION_ID) {
    return false;
  }

  // Packets with connection IDs for active connections are processed
  // immediately.
  QuicConnectionId connection_id = header.connection_id;
  SessionMap::iterator it = session_map_.find(connection_id);
  if (it != session_map_.end()) {
    DCHECK(!buffered_packets_.HasBufferedPackets(connection_id));
    // QuicSession::ProcessUdpPacket method
    it->second->ProcessUdpPacket(current_server_address_,
                                 current_client_address_, *current_packet_);
    return false;
  }

  if (FLAGS_quic_buffer_packets_after_chlo &&
      buffered_packets_.HasChloForConnection(connection_id)) {
    BufferEarlyPacket(connection_id);
    return false;
  }

  ...

}


void QuicSession::ProcessUdpPacket(const IPEndPoint& self_address,
                                   const IPEndPoint& peer_address,
                                   const QuicReceivedPacket& packet) {
  // call QuicConnection's method
  connection_->ProcessUdpPacket(self_address, peer_address, packet);
}
~~~


从这里开始，服务器的过程与客户端的相同。

当服务器接收到新的连接时，它将创建一个会话类QuicSimpleServerSession类实例。

而当服务器会话接收到新的流( 由客户端打开的流 )时，将会调用会话
QuicSimpleServerSession类的CreateIncomingDynamicStream方法。

~~~c++
QuicSpdyStream* QuicSimpleServerSession::CreateIncomingDynamicStream(
    QuicStreamId id) {
  if (!ShouldCreateIncomingDynamicStream(id)) {
    return nullptr;
  }

  QuicSpdyStream* stream = new QuicSimpleServerStream(id, this);
  ActivateStream(base::WrapUnique(stream));
  return stream;
}
~~~

此处新建了一个QuicSimpleServerStream类
\[src/net/tools/quic/quic_simple_server_stream.h&cc\]
( 其基类是QuicSpdyStream类，而QuicSpdyStream类的基类是ReliableQuicStream类 )实例。 流子类实现OnDataAvailable方法，可以在其中决定如何处理传入的数据。

~~~c++
void QuicSimpleServerStream::OnDataAvailable() {
  while (HasBytesToRead()) {
    struct iovec iov;
    if (GetReadableRegions(&iov, 1) == 0) {
      // No more data to read.
      break;
    }
    DVLOG(1) << "Processed " << iov.iov_len << " bytes for stream " << id();
    body_.append(static_cast<char*>(iov.iov_base), iov.iov_len);

    if (content_length_ >= 0 &&
        body_.size() > static_cast<uint64_t>(content_length_)) {
      DVLOG(1) << "Body size (" << body_.size() << ") > content length ("
               << content_length_ << ").";
      SendErrorResponse();
      return;
    }
    MarkConsumed(iov.iov_len);
  }
  if (!sequencer()->IsClosed()) {
    sequencer()->SetUnblocked();
    return;
  }

  // If the sequencer is closed, then all the body, including the fin, has been
  // consumed.
  OnFinRead();

  if (write_side_closed() || fin_buffered()) {
    return;
  }

  SendResponse();
}
~~~

### 1.3 数据包写入器

要处理数据包写入，必须实现QuicPacketWriter接口
\[src/net/quic/core/quic_packet_writer.h\]。 实现的主要方法是WritePacket。
它需要原始数据的缓冲区，缓冲区的长度，本地IP地址和对方的IP地址。 它负责将数据
写入网络套接字。

服务器QuicSimpleServer使用的是自定义的QuicSimpleServerPacketWriter类
\[src/net/tools/quic/quic_simple_server_packet_writer.h&cc\]
( 其基类是QuicPacketWriter类 )。

而客户端QuicSimpleClient使用的是现有的QuicChromiumPacketWriter类
\[src/net/quic/chromium/quic_chromium_packet_writer.h&cc\]
( 其基类是QuicPacketWriter类 )。

~~~c++
WriteResult QuicSimpleServerPacketWriter::WritePacket(
    const char* buffer,
    size_t buf_len,
    const IPAddress& self_address,
    const IPEndPoint& peer_address,
    PerPacketOptions* options) {
  scoped_refptr<StringIOBuffer> buf(
      new StringIOBuffer(std::string(buffer, buf_len)));
  DCHECK(!IsWriteBlocked());
  int rv;
  if (buf_len <= static_cast<size_t>(std::numeric_limits<int>::max())) {
    rv = socket_->SendTo(
        buf.get(), static_cast<int>(buf_len), peer_address,
        base::Bind(&QuicSimpleServerPacketWriter::OnWriteComplete,
                   weak_factory_.GetWeakPtr()));
  } else {
    rv = ERR_MSG_TOO_BIG;
  }
  WriteStatus status = WRITE_STATUS_OK;
  if (rv < 0) {
    if (rv != ERR_IO_PENDING) {
      UMA_HISTOGRAM_SPARSE_SLOWLY("Net.QuicSession.WriteError", -rv);
      status = WRITE_STATUS_ERROR;
    } else {
      status = WRITE_STATUS_BLOCKED;
      write_blocked_ = true;
    }
  }
  return WriteResult(status, rv);
}
~~~

数据包写入器在QuicSimpleClient和QuicSimpleServer中创建。 然后将其传递给QuicConnection，并将其用于将QUIC数据包写入网络套接字。

### 1.4 数据包读取器

数据包读取，使用的是QuicPacketReader接口
\[src/net/tools/quic/quic_packet_reader.h&cc\]。 实现的主要方法是Initialize，
初始化读取的缓冲区，以及ReadAndDispatchPackets方法，读取并分发数据包给实现
ProcessPacketInterface的类进行处理。

~~~c++
void QuicPacketReader::Initialize() {
#if MMSG_MORE
  // Zero initialize uninitialized memory.
  memset(mmsg_hdr_, 0, sizeof(mmsg_hdr_));

  for (int i = 0; i < kNumPacketsPerReadMmsgCall; ++i) {
    packets_[i].iov.iov_base = packets_[i].buf;
    packets_[i].iov.iov_len = kMaxPacketSize;
    memset(&packets_[i].raw_address, 0, sizeof(packets_[i].raw_address));
    memset(packets_[i].cbuf, 0, sizeof(packets_[i].cbuf));
    memset(packets_[i].buf, 0, sizeof(packets_[i].buf));

    msghdr* hdr = &mmsg_hdr_[i].msg_hdr;
    hdr->msg_name = &packets_[i].raw_address;
    hdr->msg_namelen = sizeof(sockaddr_storage);
    hdr->msg_iov = &packets_[i].iov;
    hdr->msg_iovlen = 1;

    hdr->msg_control = packets_[i].cbuf;
    hdr->msg_controllen = QuicSocketUtils::kSpaceForCmsg;
  }
#endif
}

bool QuicPacketReader::ReadAndDispatchPackets(
    int fd,
    int port,
    const QuicClock& clock,
    ProcessPacketInterface* processor,
    QuicPacketCount* packets_dropped) {
#if MMSG_MORE
  return ReadAndDispatchManyPackets(fd, port, clock, processor,
                                    packets_dropped);
#else
  return ReadAndDispatchSinglePacket(fd, port, clock, processor,
                                     packets_dropped);
#endif
}
~~~

数据包写入器在QuicSimpleClient和QuicSimpleServer中创建。 然后将其传递给
QuicConnection，并将其用于将QUIC数据包写入网络套接字。

### 1.5 Epoll服务器

为了异步处理网络连接，可以使用Epoll服务器。Epoll服务器的实现已经存在于
net/tools/epoll_server文件夹中的chromium源代码中。quic_simple_server_bin.cc中
没有使用Epoll服务器。

当然，在quic_server_bin.cc中使用了Epoll服务器。

---

## 2 子类

对于每个服务器和客户端，您至少需要子类化QuicSession类和可能的QuicSpdyStream类。

### 2.1 QuicSession的子类

所有QuicSession子类必须实现以下方法，因为它们在基类中是纯虚的。

* GetCryptoStream \-
返回保留的加密流。 此流必须在该子类中创建。

* CreateIncomingDynamicStream \-
如QUIC服务器程序部分中所述，当创建新流时，将调用此方法，因为客户端打开了一个新流。 如果你不想创建传入流，可以只返回nullptr。

* CreateOutgoingDynamicStream \-
与CreateIncominDataStream相同，但打开的是自己的输出流（而不是对等体）。 注意，
如果你创建一个流的实例，你必须调用含有该流会话的ActivateStream方法，
否则流将永远不会接收任何数据包！


当创建一个新的QuicSpdyStream类(或其自定义的子类）时，会调用最后两个方法。
通过实现这些方法，会话可以使用自定义的QuicSpdyStream子类。 具体可以查看QUIC
服务器程序或客户端程序的示例。

客户端会话类QuicClientSession继承自QuicClientSessionBase类
( 其基类是QuicSpdySession类和QuicCryptoClientStream::ProofHandler类，
而QuicSpdySession的基类是QuicSession类 )。 它必须实现OnProofValid方法和
OnProofVerifyDetailsAvailable方法，因为它们在基类中是纯虚的。 如果不想使用
安全连接，可以是以空操作来实现。如果需要使用安全连接，这两个方法都必须实现，
还必须实现加密流(crypto stream)的创建和加密协商的启动。

~~~c++
void QuicClientSession::Initialize() {
  crypto_stream_ = CreateQuicCryptoStream();
  QuicClientSessionBase::Initialize();
}
~~~

服务器会话类QuicSimpleServerSession继承自QuicServerSessionBase类
( 其基类是QuicSpdySession类和QuicCryptoClientStream::ProofHandler类，
而QuicSpdySession的基类是QuicSession类 )。 与客户端会话类相同，也必须实现
加密流(crypto stream)的处理。

还可以重写CreateIncomingDynamicStream方法并使它返回一个你自己的流类的实例。

~~~c++
QuicSpdyStream* QuicSimpleServerSession::CreateIncomingDynamicStream(
    QuicStreamId id) {
  if (!ShouldCreateIncomingDynamicStream(id)) {
    return nullptr;
  }
  QuicSpdyStream* stream = new QuicSpdyClientStream(id, this);
  ActivateStream(base::WrapUnique(stream));
  return stream;
}
~~~

### 2.2 Stream的子类

在客户端中，可以使用现有的QuicSpdyStream类。此处使用的是QuicSpdyClientStream
来继承QuicSpdyStream类。

在服务器中必须要自定义新的类QuicSimpleServerStream来继承QuicSpdyStream类。
在流类创建时，必须调用sequencer()->FlushBufferedFrames()来取消序列化程序的阻塞。
否则，直到SPDY请求的头部被全部接收，它才会传递新数据到流。 然后，覆盖
OnDataAvailable方法，它将接收此流的所有数据。

---

## 3 附录

### 3.1 部分核心类

实现QUIC不同方面的各种类。

#### QuicConnection类

QuicConnection类处理QUIC客户端或服务器的成帧。 它提供了SendStreamData方法
(由QuicSession调用)发送流数据。 它反过来使用QuicPacketGenerator来生成
QuicFrames。 QuicConnection还实现了QuicPacketGenerator :: DelegateInterface，
并分配给QuicPacketGenerator作为委托。然后QuicPacketGenerator会调用
QuicConnection的OnSerializedPacket方法。 最后，使用QuicPacketWriter
将这些帧写入WritePacketInner的下层连接中。

#### QuicSession类

QuicSession类是一个基类，具体的会话类必须从该基类继承。 它的主要功能是将传入的
数据分派到正确的QUIC流中。 它还拥有QuicConnection，用于通过连接发送数据。
因此，它表示QUIC连接(包括多个流)，是真实网络连接的抽象。 QUIC流调用WritevData
方法来发送数据。 反过来，QuicConnection将调用QuicConnectionVisitorInterface的
方法来通知会话新数据包的到达或连接的更改。

#### ReliableQuicStream类

ReliableQuicStream类是一个基类，用以实现QUIC流。它定义了QUIC流类必须满足的接口。
它还实现流的基本逻辑，例如流控制，帧排序，流处理，连接复位或关闭和缓存数据写入。
然后，一个完整的QUIC流类只需要实现ProcessRawData方法和EffectivePriority方法。

#### QuicSpdyStream类

QuicSpdyStream类实现传输SPDY请求的QUIC流。头部将在由会话管理的专用头部流
QuicHeadersStream传输。通过调用OnStreamHeaderList方法和OnStreamHeadersPriority
方法来调度头部的传送。在初始化时，它会阻塞QuicStreamSequencer类，直到接收到所有头部。

#### QuicStreamSequencer类

QuicStreamSequencer类用来缓冲帧，直到它们可以传递到下一层。 包括重复帧的检查，
帧排序，以便将数据排序并检查错误。

#### QuicPacketCreator类

QuicPacketCreator类处理帧和数据包的生成。 它可以缓冲帧以构建由多个帧组成的较大
分组，并且还可以生成帧的FEC分组。 它由QuicPacketGenerator类使用以生成数据包。

#### QuicPacketGenerator类

QuicPacketGenerator类由QuicConnection类使用以生成和发送数据包。 它使用
QuicPacketCreator类来构建帧和数据包。 当分组准备就绪时，它调用委托给自己的
OnSerializedPacket方法。

#### QuicFramer类

QuicFramer类解析和构建QUIC数据包。 它通过ProcessPacket方法接收数据，并调用
QuicFramerVisitorInterface的方法来通知QuicConnection新数据包的到达。

#### QuicHeadersStream类

QuicHeadersStream类传输QuicDataConnection的带外SPDY头。

### 3.2 部分接口

一些接口(而不是具体类)。

#### QuicPacketWriter

PacketWriter接口定义将由QuicConnection调用以发送数据包的方法。 它还定义了一些
方法来确定套接字是否被阻塞。 这些方法必须在使用QUIC的应用程序实现。

#### QuicPacketGenerator::DelegateInterface

QuicPacketGenerator :: DelegateInterface接口定义当新数据包可用时
QuicPacketGenerator类将调用的方法。 它由QuicConnection类实现。

#### QuicFrameVisitorInterface

QuicFrameVisitorInterface接口定义了QuicFramer在处理新的QUIC数据包时调用的方法。
它也由QuicConnection类实现。

#### QuicConnectionHelperInterface

QuicConnectionHelperInterface接口定义了QuicConnection类使用的一些方法来获取时钟，
获取随机值的源或设置定时器。

#### QuicConnectionVisitorInterface

QuicConnectionVisitorInterface接口定义在接收帧或发生其他事件时由QuicConnection类
调用的方法。 它由QuicSession类实现。 此接口的OnStreamFrame方法用于将流帧从连接
（connection）传递到会话（session）。

## 参考文献
- [proto-quic committed as Updating to 56.0.2912.0 \(#15\)](https://github.com/google/proto-quic/commit/0db5f234b86699c182394bb261ee226ab800b60f)
- [Write your own QUIC application](https://github.com/maufl/quic_toy/blob/master/howto.markdown)

---
The End.

zhlinh

Email: zhlinhng@gmail.com

2017-01-21
