---
title: Network Library 03) Recv-Send Buffer
categories: RookissGameServer
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part4:  게임 서버> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. Recv Buffer**

TCP에서는 패킷이 전송했던 크기와 다르게 나누어져서 오거나 뭉쳐서 오는 경우가 생긴다. 따라서 원형 큐 형식의 링 버퍼를 사용해서 송수신을 처리하고, 한 묶음 단위의 패킷 별로 뽑아서 쓸 수 있도록 하는 것이 좋다. 이 예제에서는 지금까지 읽은 위치와 데이터를 마지막으로 쓴 위치를 저장해서 이를 바탕으로 링 버퍼를 관리하고 있다. 또 데이터를 쓰는 속도에 비해 읽는 속도가 느려서 링 버퍼가 부족할 경우를 대비에 요청 받은 버퍼 크기의 10배 되는 크기를 capacity로 갖는다. 

**RecvBuffer.h**

```c++
#pragma once

class RecvBuffer
{
	enum { BUFFER_COUNT = 10 };

public:
	RecvBuffer(int32 bufferSize);
	~RecvBuffer();

	void			Clean();
	bool			OnRead(int32 numOfBytes);
	bool			OnWrite(int32 numOfBytes);

	BYTE*			ReadPos() { return &_buffer[_readPos]; }
	BYTE*			WritePos() { return &_buffer[_writePos]; }
	int32			DataSize() { return _writePos - _readPos; }
	int32			FreeSize() { return _capacity - _writePos; }

private:
	int32			_capacity = 0;
	int32			_bufferSize = 0;
	int32			_readPos = 0;
	int32			_writePos = 0;
	Vector<BYTE>	_buffer;
};
```

**RecvBuffer.cpp**

```c++
#include "pch.h"
#include "RecvBuffer.h"

RecvBuffer::RecvBuffer(int32 bufferSize) : _bufferSize(bufferSize)
{
	_capacity = bufferSize * BUFFER_COUNT;
	_buffer.resize(_capacity);
}

RecvBuffer::~RecvBuffer()
{
}

void RecvBuffer::Clean()
{
	int32 dataSize = DataSize();
	if (dataSize == 0)
	{
		// 읽기 커서, 쓰기 커서 위치가 같아지면 리셋.
		_readPos = _writePos = 0;
	}
	else
	{
		// 여유 공간이 버퍼 1개 크기 미만이면 데이터를 앞으로 당긴다.
		if (FreeSize() < _bufferSize)
		{
			::memcpy(&_buffer[0], &_buffer[_readPos], dataSize);
			_readPos = 0;
			_writePos = dataSize;
		}
	}
}

bool RecvBuffer::OnRead(int32 numOfBytes)
{
	if (numOfBytes > DataSize())
		return false;

	_readPos += numOfBytes;
	return true;
}

bool RecvBuffer::OnWrite(int32 numOfBytes)
{
	if (numOfBytes > FreeSize())
		return false;

	_writePos += numOfBytes;
	return true;
}
```

지금까지의 예제에서는 고정 길이 배열로 Buffer를 관리했는데, Session 클래스의 Buffer 멤버를 위의 RecvBuffer 객체로 교체한다. 이후 RecvBuffer가 쓰이는 멤버 함수들을 아래와 같이 수정한다. 

**Session.cpp**
```c++
#include "pch.h"
#include "Session.h"
#include "SocketUtils.h"
#include "Service.h"

Session::Session() : _recvBuffer(BUFFER_SIZE)
{
	_socket = SocketUtils::CreateSocket();
}

void Session::RegisterRecv()
{
	if (IsConnected() == false)
		return;

	_recvEvent.Init();
	_recvEvent.owner = shared_from_this(); // ADD_REF

	WSABUF wsaBuf;
	wsaBuf.buf = reinterpret_cast<char*>(_recvBuffer.WritePos());
	wsaBuf.len = _recvBuffer.FreeSize();

	DWORD numOfBytes = 0;
	DWORD flags = 0;
	if (SOCKET_ERROR == ::WSARecv(_socket, &wsaBuf, 1, OUT &numOfBytes, OUT &flags, &_recvEvent, nullptr))
	{
		int32 errorCode = ::WSAGetLastError();
		if (errorCode != WSA_IO_PENDING)
		{
			HandleError(errorCode);
			_recvEvent.owner = nullptr; // RELEASE_REF
		}
	}
}

void Session::ProcessRecv(int32 numOfBytes)
{
	_recvEvent.owner = nullptr; // RELEASE_REF

	if (numOfBytes == 0)
	{
		Disconnect(L"Recv 0");
		return;
	}

	if (_recvBuffer.OnWrite(numOfBytes) == false)
	{
		Disconnect(L"OnWrite Overflow");
		return;
	}

	int32 dataSize = _recvBuffer.DataSize();
	int32 processLen = OnRecv(_recvBuffer.ReadPos(), dataSize); 
	if (processLen < 0 || dataSize < processLen || _recvBuffer.OnRead(processLen) == false)
	{
		Disconnect(L"OnRead Overflow");
		return;
	}
	
	// 커서 정리
	_recvBuffer.Clean();

	// 수신 등록
	RegisterRecv();
}
```

<br/> 

# **2. Send Buffer**

Recv Buffer와 달리 Send Buffer는 멀티스레드에서 문제가 생길 수 있으므로 enable_shared_from_this를 상속 받아서 동기화 작업을 해주어야 한다. 아래는 기본적인 SendBuffer 구현 예제인데, Session에서는 이를 그대로 사용하는 대신 SendBuffer 포인터의 Queue, 그리고 atomic bool 값과 LOCK 을 이용해서 송신을 처리한다. 

**SendBuffer.h**
```c++
#pragma once

class SendBuffer : enable_shared_from_this<SendBuffer>
{
public:
	SendBuffer(int32 bufferSize);
	~SendBuffer();

	BYTE* Buffer() { return _buffer.data(); }
	int32 WriteSize() { return _writeSize; }
	int32 Capacity() { return static_cast<int32>(_buffer.size()); }

	void CopyData(void* data, int32 len);

private:
	Vector<BYTE>	_buffer;
	int32			_writeSize = 0;
};
```

**SendBuffer.cpp**
```c++
#include "pch.h"
#include "SendBuffer.h"

SendBuffer::SendBuffer(int32 bufferSize)
{
	_buffer.resize(bufferSize);
}

SendBuffer::~SendBuffer()
{
}

void SendBuffer::CopyData(void* data, int32 len)
{
	ASSERT_CRASH(Capacity() >= len);
	::memcpy(_buffer.data(), data, len);
	_writeSize = len;
}
```

**Session.h**
```c++
class Session : public IocpObject
{
	enum
	{
		BUFFER_SIZE = 0x10000, // 64KB
	};

private:
	USE_LOCK;

							/* 수신 관련 */
	RecvBuffer				_recvBuffer;

							/* 송신 관련 */
	Queue<SendBufferRef>	_sendQueue;
	Atomic<bool>			_sendRegistered = false;

private:
						/* IocpEvent 재사용 */
	ConnectEvent		_connectEvent;
	DisconnectEvent		_disconnectEvent;
	RecvEvent			_recvEvent;
	SendEvent			_sendEvent;
};
```

**Session.cpp**
```c++
#include "pch.h"
#include "Session.h"
#include "SocketUtils.h"
#include "Service.h"

void Session::Send(SendBufferRef sendBuffer)
{
	// 현재 RegisterSend가 걸리지 않은 상태라면, 걸어준다
	WRITE_LOCK;

	_sendQueue.push(sendBuffer);

	/*if (_sendRegistered == false)
	{
		_sendRegistered = true;
		RegisterSend();
	}*/
	
	if (_sendRegistered.exchange(true) == false)
		RegisterSend();
}

void Session::RegisterSend()
{
	if (IsConnected() == false)
		return;

	_sendEvent.Init();
	_sendEvent.owner = shared_from_this(); // ADD_REF

	// 보낼 데이터를 sendEvent에 등록
	{
		WRITE_LOCK;

		int32 writeSize = 0;
		while (_sendQueue.empty() == false)
		{
			SendBufferRef sendBuffer = _sendQueue.front();

			writeSize += sendBuffer->WriteSize();
			// TODO : 예외 체크

			_sendQueue.pop();
			_sendEvent.sendBuffers.push_back(sendBuffer);
		}
	}

	// Scatter-Gather (흩어져 있는 데이터들을 모아서 한 방에 보낸다)
	Vector<WSABUF> wsaBufs;
	wsaBufs.reserve(_sendEvent.sendBuffers.size());
	for (SendBufferRef sendBuffer : _sendEvent.sendBuffers)
	{
		WSABUF wsaBuf;
		wsaBuf.buf = reinterpret_cast<char*>(sendBuffer->Buffer());
		wsaBuf.len = static_cast<LONG>(sendBuffer->WriteSize());
		wsaBufs.push_back(wsaBuf);
	}

	DWORD numOfBytes = 0;
	if (SOCKET_ERROR == ::WSASend(_socket, wsaBufs.data(), static_cast<DWORD>(wsaBufs.size()), OUT &numOfBytes, 0, &_sendEvent, nullptr))
	{
		int32 errorCode = ::WSAGetLastError();
		if (errorCode != WSA_IO_PENDING)
		{
			HandleError(errorCode);
			_sendEvent.owner = nullptr; // RELEASE_REF
			_sendEvent.sendBuffers.clear(); // RELEASE_REF
			_sendRegistered.store(false);
		}
	}
}

void Session::ProcessSend(int32 numOfBytes)
{
	_sendEvent.owner = nullptr; // RELEASE_REF
	_sendEvent.sendBuffers.clear(); // RELEASE_REF

	if (numOfBytes == 0)
	{
		Disconnect(L"Send 0");
		return;
	}

	// 컨텐츠 코드에서 재정의
	OnSend(numOfBytes);

	WRITE_LOCK;
	if (_sendQueue.empty())
		_sendRegistered.store(false);
	else
		RegisterSend();
}
```

<br/> 

# **3. Send Buffer Pooling**

Send Buffer를 더 효율적으로 사용하기 위해 pooling을 적용할 수 있다. 큰 크기의 Send Buffer Chunk를 일단 할당 받은 뒤, 필요할 때마다 버퍼를 분배함으로써 매번 할당하는 데 소요되는 리소스를 절약하는 것이다. 이전 강의에서 구현했던 ObjectPool을 활용한다.

**SendBuffer.h**
```c++
#pragma once

class SendBufferChunk;

class SendBuffer
{
public:
	SendBuffer(SendBufferChunkRef owner, BYTE* buffer, int32 allocSize);
	~SendBuffer();

	BYTE*		Buffer() { return _buffer; }
	int32		WriteSize() { return _writeSize; }
	void		Close(uint32 writeSize);

private:
	BYTE*				_buffer;
	uint32				_allocSize = 0;
	uint32				_writeSize = 0;
	SendBufferChunkRef	_owner;
};

class SendBufferChunk : public enable_shared_from_this<SendBufferChunk>
{
	enum
	{
		SEND_BUFFER_CHUNK_SIZE = 6000
	};

public:
	SendBufferChunk();
	~SendBufferChunk();

	void				Reset();
	SendBufferRef		Open(uint32 allocSize);
	void				Close(uint32 writeSize);

	bool				IsOpen() { return _open; }
	BYTE*				Buffer() { return &_buffer[_usedSize]; }
	uint32				FreeSize() { return static_cast<uint32>(_buffer.size()) - _usedSize; }

private:
	Array<BYTE, SEND_BUFFER_CHUNK_SIZE>		_buffer = {};
	bool									_open = false;
	uint32									_usedSize = 0;
};

class SendBufferManager
{
public:
	SendBufferRef		Open(uint32 size);

private:
	SendBufferChunkRef	Pop();
	void				Push(SendBufferChunkRef buffer);

	static void			PushGlobal(SendBufferChunk* buffer);

private:
	USE_LOCK;
	Vector<SendBufferChunkRef> _sendBufferChunks;
};
```

**SendBuffer.cpp**
```c++
#include "pch.h"
#include "SendBuffer.h"

SendBuffer::SendBuffer(SendBufferChunkRef owner, BYTE* buffer, int32 allocSize)
	: _owner(owner), _buffer(buffer), _allocSize(allocSize)
{
}

SendBuffer::~SendBuffer()
{
}

void SendBuffer::Close(uint32 writeSize)
{
	ASSERT_CRASH(_allocSize >= writeSize);
	_writeSize = writeSize;
	_owner->Close(writeSize);
}

SendBufferChunk::SendBufferChunk()
{
}

SendBufferChunk::~SendBufferChunk()
{
}

void SendBufferChunk::Reset()
{
	_open = false;
	_usedSize = 0;
}

SendBufferRef SendBufferChunk::Open(uint32 allocSize)
{
	ASSERT_CRASH(allocSize <= SEND_BUFFER_CHUNK_SIZE);
	ASSERT_CRASH(_open == false);

	if (allocSize > FreeSize())
		return nullptr;

	_open = true;
	return ObjectPool<SendBuffer>::MakeShared(shared_from_this(), Buffer(), allocSize);
}

void SendBufferChunk::Close(uint32 writeSize)
{
	ASSERT_CRASH(_open == true);
	_open = false;
	_usedSize += writeSize;
}

SendBufferRef SendBufferManager::Open(uint32 size)
{
	if (LSendBufferChunk == nullptr)
	{
		LSendBufferChunk = Pop(); // WRITE_LOCK
		LSendBufferChunk->Reset();
	}		

	ASSERT_CRASH(LSendBufferChunk->IsOpen() == false);

	// 다 썼으면 버리고 새거로 교체
	if (LSendBufferChunk->FreeSize() < size)
	{
		LSendBufferChunk = Pop(); // WRITE_LOCK
		LSendBufferChunk->Reset();
	}

	cout << "FREE : " << LSendBufferChunk->FreeSize() << endl;

	return LSendBufferChunk->Open(size);
}

SendBufferChunkRef SendBufferManager::Pop()
{
	{
		WRITE_LOCK;
		if (_sendBufferChunks.empty() == false)
		{
			SendBufferChunkRef sendBufferChunk = _sendBufferChunks.back();
			_sendBufferChunks.pop_back();
			return sendBufferChunk;
		}
	}

	return SendBufferChunkRef(xnew<SendBufferChunk>(), PushGlobal);
}

void SendBufferManager::Push(SendBufferChunkRef buffer)
{
	WRITE_LOCK;
	_sendBufferChunks.push_back(buffer);
}

void SendBufferManager::PushGlobal(SendBufferChunk* buffer)
{
	GSendBufferManager->Push(SendBufferChunkRef(buffer, PushGlobal));
}
```

<br/> 

# **4. Packet Session**

데이터가 적절한 단위로 모였는지 판단하기 위해 패킷 헤더를 만들어서 구분해야 한다. 아래 예제는 Session의 자식 클래스인 Packet Session을 통해 송수신 데이터를 패킷 단위로 처리하고 있다. buffer를 Packet Header로 캐스팅 하여 예정된 길이만큼 도착했는지 확인하고, 도착한 경우 OnRecvPacket로 통지한다. GameSession도 기존의 Session 대신 이 Packet Session을 상속 받아서 송수신 단위 대신 패킷 단위로 데이터를 받을 수 있도록 수정하였다. 참고로 GameServer 예제의 경우 버퍼에서 데이터를 적절하게 불러오는 작업을 메인 스레드에서 진행하고 있는데, 이렇게 해당 작업을 Network IO와 분리하면 CPU를 더 효율적으로 활용할 수 있다.  

**Session.h**
```c++
struct PacketHeader
{
	uint16 size;
	uint16 id;
};

class PacketSession : public Session
{
public:
	PacketSession();
	virtual ~PacketSession();

	PacketSessionRef GetPacketSessionRef()
	{
		return static_pointer_cast<PacketSession>(shared_from_this());
	}

protected:
	virtual int32 OnRecv(BYTE* buffer, int32 len) sealed;
	virtual int32 OnRecvPacket(BYTE* buffer, int32 len) abstract;

};
```

**Session.cpp**
```c++
int32 PacketSession::OnRecv(BYTE* buffer, int32 len)
{
	int32 processLen = 0;

	while (true)
	{
		int32 dataSize = len - processLen;
		if (dataSize < sizeof(PacketHeader))
			break;
		
		PacketHeader header = *((PacketHeader*)&buffer[processLen]);
		if (dataSize < header.size)
			break;

		OnRecvPacket(&buffer[processLen], header.size);
		processLen += header.size;
	}
	return processLen;
}

```

**GameServer.cpp (Main)**

```c++
class GameSession : public PacketSession
{
public:
	// ...
	virtual int32 OnRecvPacket(BYTE* buffer, int32 len)
	{
		PacketHeader header = *((PacketHeader*)&buffer[0]);
		cout << "Size = " << header.size << endl; 
		return header.size;
	}
};

int main()
{
	ServerServiceRef service = MakeShared<ServerService>(
		NetAddress(L"127.0.0.1", 7777),
		MakeShared<IocpCore>(),
		MakeShared<GameSession>, // TODO : SessionManager 등
		100);

	ASSERT_CRASH(service->Start());

	for (int32 i = 0; i < 5; i++)
	{
		GThreadManager->Launch([=]()
			{
				while (true)
				{
					service->GetIocpCore()->Dispatch();
				}				
			});
	}	

	char sendData[] = "Hello World";
	while (1)
	{
		SendBufferRef sendBuffer = GSendBufferManager->Open(4096);
		
		BYTE* buffer = sendBuffer->Buffer();
		((PacketHeader*)buffer)->size = (sizeof(PacketHeader) + sizeof(sendData));
		((PacketHeader*)buffer)->id = 1;
		
		::memcpy(&buffer[4], sendData, sizeof(sendData));
		sendBuffer->Close(sizeof(PacketHeader) + sizeof(sendData));

		GSessionManager.Broadcast(sendBuffer);
		this_thread::sleep_for(250ms);
	}
	GThreadManager->Join();
}
```

**DummyClient.cpp (Main)**
```c++
class ServerSession : public PacketSession
{
public:
	// ...

	virtual int32 OnRecvPacket(BYTE* buffer, int32 len) override
	{
		PacketHeader header = *((PacketHeader*)&buffer[0]);
		cout << "Size = " << header.size << endl;

		char recvBuffer[4096];
		::memcpy(recvBuffer, &buffer[4], header.size - sizeof(PacketHeader));
		cout << recvBuffer << endl;

		return len;
	}
};

char sendData[] = "Hello World";

int main()
{
	this_thread::sleep_for(1s);

	ClientServiceRef service = MakeShared<ClientService>(
		NetAddress(L"127.0.0.1", 7777),
		MakeShared<IocpCore>(),
		MakeShared<ServerSession>, // TODO : SessionManager 등
		5);

	ASSERT_CRASH(service->Start());

	for (int32 i = 0; i < 2; i++)
	{
		GThreadManager->Launch([=]()
			{
				while (true)
				{
					service->GetIocpCore()->Dispatch();
				}
			});
	}

	GThreadManager->Join();
}
```

# **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버, Rookiss
