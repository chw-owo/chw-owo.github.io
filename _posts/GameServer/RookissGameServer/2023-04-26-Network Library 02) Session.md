---
title: Network Library 02) Session
categories: RookissGameServer
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part4:  게임 서버> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. Service**

Session들을 들고 있으면서 서비스 종류에 따라 적합하게 관리할 Service 클래스를 만든다. 이를 서비스 타입 (클라이언트/서버)로 나누어 상속함으로써, 자식 객체 하나로 각 서비스 타입에 맞는 작업을 처리할 수 있도록 구현되었다. 

**Service.h**

```c++
#pragma once
#include "NetAddress.h"
#include "IocpCore.h"
#include "Listener.h"
#include <functional>

enum class ServiceType : uint8
{
	Server,
	Client
};

using SessionFactory = function<SessionRef(void)>;

class Service : public enable_shared_from_this<Service>
{
public:
	Service(ServiceType type, NetAddress address, IocpCoreRef core, SessionFactory factory, int32 maxSessionCount = 1);
	virtual ~Service();

	virtual bool		Start() abstract;
	bool				CanStart() { return _sessionFactory != nullptr; }

	virtual void		CloseService();
	void				SetSessionFactory(SessionFactory func) { _sessionFactory = func; }

	SessionRef			CreateSession();
	void				AddSession(SessionRef session);
	void				ReleaseSession(SessionRef session);
	int32				GetCurrentSessionCount() { return _sessionCount; }
	int32				GetMaxSessionCount() { return _maxSessionCount; }

public:
	ServiceType			GetServiceType() { return _type; }
	NetAddress			GetNetAddress() { return _netAddress; }
	IocpCoreRef&		GetIocpCore() { return _iocpCore; }

protected:
	USE_LOCK;
	ServiceType			_type;
	NetAddress			_netAddress = {};
	IocpCoreRef			_iocpCore;

	Set<SessionRef>		_sessions;
	int32				_sessionCount = 0;
	int32				_maxSessionCount = 0;
	SessionFactory		_sessionFactory;
};

class ClientService : public Service
{
public:
	ClientService(NetAddress targetAddress, IocpCoreRef core, SessionFactory factory, int32 maxSessionCount = 1);
	virtual ~ClientService() {}

	virtual bool	Start() override;
};

class ServerService : public Service
{
public:
	ServerService(NetAddress targetAddress, IocpCoreRef core, SessionFactory factory, int32 maxSessionCount = 1);
	virtual ~ServerService() {}

	virtual bool	Start() override;
	virtual void	CloseService() override;

private:
	ListenerRef		_listener = nullptr;
};
```

서비스는 멤버 함수로 Service에 대한 Start, Close 함수와 함께 세션 처리를 위한 Create/Add/ReleaseSession 함수들을 갖는다. 멤버 변수로는 타입, 주소, IocpCore 포인터와 함께 세션 Set, 세션 개수 등을 갖는다. 또 Server Service의 경우 Listen 처리를 위한 IocpObject, Listener의 포인터를 추가로 갖는다. 

**Service.cpp**

```c++
#include "pch.h"
#include "Service.h"
#include "Session.h"
#include "Listener.h"

Service::Service(ServiceType type, NetAddress address, IocpCoreRef core, SessionFactory factory, int32 maxSessionCount)
	: _type(type), _netAddress(address), _iocpCore(core), _sessionFactory(factory), _maxSessionCount(maxSessionCount)
{

}

Service::~Service()
{
}

void Service::CloseService()
{
	// TODO
}

SessionRef Service::CreateSession()
{
	SessionRef session = _sessionFactory();
	session->SetService(shared_from_this());

	if (_iocpCore->Register(session) == false)
		return nullptr;

	return session;
}

void Service::AddSession(SessionRef session)
{
	WRITE_LOCK;
	_sessionCount++;
	_sessions.insert(session);
}

void Service::ReleaseSession(SessionRef session)
{
	WRITE_LOCK;
	ASSERT_CRASH(_sessions.erase(session) != 0);
	_sessionCount--;
}

ClientService::ClientService(NetAddress targetAddress, IocpCoreRef core, SessionFactory factory, int32 maxSessionCount)
	: Service(ServiceType::Client, targetAddress, core, factory, maxSessionCount)
{
}

bool ClientService::Start()
{
	if (CanStart() == false)
		return false;

	const int32 sessionCount = GetMaxSessionCount();
	for (int32 i = 0; i < sessionCount; i++)
	{
		SessionRef session = CreateSession();
		if (session->Connect() == false)
			return false;
	}

	return true;
}

ServerService::ServerService(NetAddress address, IocpCoreRef core, SessionFactory factory, int32 maxSessionCount)
	: Service(ServiceType::Server, address, core, factory, maxSessionCount)
{
}

bool ServerService::Start()
{
	if (CanStart() == false)
		return false;

	_listener = MakeShared<Listener>();
	if (_listener == nullptr)
		return false;

	ServerServiceRef service = static_pointer_cast<ServerService>(shared_from_this());
	if (_listener->StartAccept(service) == false)
		return false;

	return true;
}

void ServerService::CloseService()
{
	// TODO

	Service::CloseService();
}
```

Start 함수는 Client의 경우 현재 세션 개수가 최대 세션 개수 이하일 때 세션을 생성하는 기능을, Server의 경우 Listener를 생성하여 Accept하는 기능을 수행한다. CreateSession은 Session 생성, Session-Service 연결, IocpCore 등록 작업을 수행하며, Add/Release는 insert/erase 및 sessionCount 관리를 수행한다. 

<br/> 

# **2. Main**

Server, Client 측에서 각각 세션을 생성하여 이를 적절한 Service 객체에 연결한다. 

**GameServer.cpp**

```c++
#include "pch.h"
#include "ThreadManager.h"
#include "Service.h"
#include "Session.h"

class GameSession : public Session
{
public:
	~GameSession()
	{
		cout << "~GameSession" << endl;
	}

	virtual int32 OnRecv(BYTE* buffer, int32 len) override
	{
		// Echo
		cout << "OnRecv Len = " << len << endl;
		Send(buffer, len);
		return len;
	}

	virtual void OnSend(int32 len) override
	{ 
		cout << "OnSend Len = " << len << endl;
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

	GThreadManager->Join();
}
```

Session을 GameSession으로 상속 받아서 OnRecv, OnSend를 오버라이딩한다. OnRecv, OnSend는 콘텐츠 단에 작업이 이루어졌음을 통지하는 용도의 함수들로, Server 예제에서는 각각 recv 길이를 출력하고 Send()를 호출, send 한 길이를 출력하는 용도로 오버라이딩 되고 있다. 

이후 해당 세션과 IocpCore의 shared_ptr을 넣고 ServerService를 생성한다. 이를 바탕으로 Start()를 실행하면 위에서 정의한 Server Service의 동작들이 자동으로 실행되며, 이후 IocpCore에게 반복해서 Dispatch 하도록 시킨다.

**DummyClient.cpp**

```c++
#include "pch.h"
#include "ThreadManager.h"
#include "Service.h"
#include "Session.h"

char sendBuffer[] = "Hello World";

class ServerSession : public Session
{
public:
	~ServerSession()
	{
		cout << "~ServerSession" << endl;
	}

	virtual void OnConnected() override
	{
		cout << "Connected To Server" << endl;
		Send((BYTE*)sendBuffer, sizeof(sendBuffer));
	}

	virtual int32 OnRecv(BYTE* buffer, int32 len) override
	{
		cout << "OnRecv Len = " << len << endl;

		this_thread::sleep_for(1s);

		Send((BYTE*)sendBuffer, sizeof(sendBuffer));
		return len;
	}

	virtual void OnSend(int32 len) override
	{
		cout << "OnSend Len = " << len << endl;
	}

	virtual void OnDisconnected() override
	{
		cout << "Disconnected" << endl;
	}
};

int main()
{
	this_thread::sleep_for(1s);

	ClientServiceRef service = MakeShared<ClientService>(
		NetAddress(L"127.0.0.1", 7777),
		MakeShared<IocpCore>(),
		MakeShared<ServerSession>, // TODO : SessionManager 등
		1);

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

DummyClient는 서버에 연결하는 세션이 필요하다. 따라서 Session을 상속 받고 On- 함수들을 오버라이딩 한 ServerSession을 만든다. OnConnected는 서버에 연결되었음을 출력하고 sendBuffer의 데이터를 Send()한다. OnRecv는 1초 간 sleep 한 뒤 sendBuffer의 데이터를 Send()하며, OnSend에서는 send 한 길이를 출력한다.

이후 Server에서와 마찬가지로 해당 세션과 IocpCore의 shared_ptr을 넣고 ClientService를 생성한다. 이를 바탕으로 Start()를 실행하면 위에서 정의한 Client Service의 동작들이 자동으로 실행되며, 이후 IocpCore에게 반복해서 Dispatch 하도록 시킨다.

**IocpCore.cpp**

```c++
bool IocpCore::Dispatch(uint32 timeoutMs)
{
	DWORD numOfBytes = 0;
	ULONG_PTR key = 0;	
	IocpEvent* iocpEvent = nullptr;

	if (::GetQueuedCompletionStatus(_iocpHandle, OUT &numOfBytes, OUT &key, OUT reinterpret_cast<LPOVERLAPPED*>(&iocpEvent), timeoutMs))
	{
		IocpObjectRef iocpObject = iocpEvent->owner;
		iocpObject->Dispatch(iocpEvent, numOfBytes);
	}
	else
	{
		int32 errCode = ::WSAGetLastError();
		switch (errCode)
		{
		case WAIT_TIMEOUT:
			return false;
		default:
			// TODO : 로그 찍기
			IocpObjectRef iocpObject = iocpEvent->owner;
			iocpObject->Dispatch(iocpEvent, numOfBytes);
			break;
		}
	}

	return true;
}
```

IocpCore::Dispatch는 위와 같은 기능을 한다. 우선 iocpHandle을 바탕으로 GetQueued- 를 실행하여 처리될 수 있는 작업(iocpEvent)을 기다린다. 여기서 iocpEvent는 eventType과 owner를 추가로 지닌 OVERLAPPED의 자식 클래스이다. 처리할 작업이 생기면 해당 iocpEvent을 소유한 IocpObject에게 Dispatch를 시킨다. 

**Listener.cpp**
```c++
class Listener: public IocpObject
{
    // ...
};

void Listener::Dispatch(IocpEvent* iocpEvent, int32 numOfBytes)
{
	ASSERT_CRASH(iocpEvent->eventType == EventType::Accept);
	AcceptEvent* acceptEvent = static_cast<AcceptEvent*>(iocpEvent);
	ProcessAccept(acceptEvent);
}
```
Listener는 Accept IocpEvent를 처리하기 위한 IocpObject이다. iocpObject -> Dispatch가 호출되면 해당 iocpEvent를 acceptEvent로 캐스팅한 후 ProcessAccept를 호출한다. ProcessAccept는 에러 체크 후 세션 초기화, 세션 Connect, AcceptEx를 수행하므로 이를 통해 비동기적으로 Accept를 처리할 수 있게 된다. 

<br/> 

# **3. Session**

한번 연결이 이루어져서 Session이 등록되고 나면 자동으로 송수신을 처리할 수 있도록 Session 클래스에 연결, 송수신, 연결 해제 기능을 추가한다. 이때 Session도 IocpObject의 자식 클래스이므로 작업 가능 상태에 진입하면 IocpCore에서 Session의 Dispatch를 호출하여 적절한 작업을 수행시킬 것이다. 

**Session.h**

```c++
#pragma once
#include "IocpCore.h"
#include "IocpEvent.h"
#include "NetAddress.h"

class Service;

class Session : public IocpObject
{
	friend class Listener;
	friend class IocpCore;
	friend class Service;

public:
	Session();
	virtual ~Session();

public:
						/* 외부에서 사용 */
	void				Send(BYTE* buffer, int32 len);
	bool				Connect();
	void				Disconnect(const WCHAR* cause);

	shared_ptr<Service>	GetService() { return _service.lock(); }
	void				SetService(shared_ptr<Service> service) { _service = service; }

public:
						/* 정보 관련 */
	void				SetNetAddress(NetAddress address) { _netAddress = address; }
	NetAddress			GetAddress() { return _netAddress; }
	SOCKET				GetSocket() { return _socket; }
	bool				IsConnected() { return _connected; }
	SessionRef			GetSessionRef() { return static_pointer_cast<Session>(shared_from_this()); }

private:
						/* 인터페이스 구현 */
	virtual HANDLE		GetHandle() override;
	virtual void		Dispatch(class IocpEvent* iocpEvent, int32 numOfBytes = 0) override;

private:
						/* 전송 관련 */
	bool				RegisterConnect();
	bool				RegisterDisconnect();
	void				RegisterRecv();
	void				RegisterSend(SendEvent* sendEvent);

	void				ProcessConnect();
	void				ProcessDisconnect();
	void				ProcessRecv(int32 numOfBytes);
	void				ProcessSend(SendEvent* sendEvent, int32 numOfBytes);

	void				HandleError(int32 errorCode);

protected:
						/* 컨텐츠 코드에서 재정의 */
	virtual void		OnConnected() { }
	virtual int32		OnRecv(BYTE* buffer, int32 len) { return len; }
	virtual void		OnSend(int32 len) { }
	virtual void		OnDisconnected() { }

public:
	// TEMP
	BYTE _recvBuffer[1000];

	// Circular Buffer
	char _sendBuffer[1000];
	int32 _sendLen = 0;

private:
	weak_ptr<Service>	_service;
	SOCKET				_socket = INVALID_SOCKET;
	NetAddress			_netAddress = {};
	Atomic<bool>		_connected = false;

private:
	USE_LOCK;

	/* 수신 관련 */

	/* 송신 관련 */

private:
						/* IocpEvent 재사용 */
	ConnectEvent		_connectEvent;
	DisconnectEvent		_disconnectEvent;
	RecvEvent			_recvEvent;
};
```

세션은 멤버 변수로 Service, Socket, Address, Connect 여부 bool 값 그리고 재사용을 위한 IocpEvent 객체들을 갖는다. 위 예제에서는 송수신용 버퍼도 임시로 갖고 있으나 이는 테스트를 위한 것이며 이후에 멀티스레드 환경에 적합한 형태로 수정될 예정이다.  

멤버 함수로는 크게 외부 호출용 함수 Connect, Send, Disconnect / 내부 처리용 함수 RegisterConnect, RegisterSend, RegisterRecv, RegisterDisconnect / 컨텐츠 코드에서 이벤트 발생을 인지하고 적절한 작업을 수행하기 위한 함수 OnConnect, OnSend, OnRecv, OnDisconnect로 구성된다.


**Session.cpp**

```c++
#include "pch.h"
#include "Session.h"
#include "SocketUtils.h"
#include "Service.h"

Session::Session()
{
	_socket = SocketUtils::CreateSocket();
}

Session::~Session()
{
	SocketUtils::Close(_socket);
}

void Session::Send(BYTE* buffer, int32 len)
{
	// TEMP
	SendEvent* sendEvent = xnew<SendEvent>();
	sendEvent->owner = shared_from_this(); // ADD_REF
	sendEvent->buffer.resize(len);
	::memcpy(sendEvent->buffer.data(), buffer, len);

	WRITE_LOCK;
	RegisterSend(sendEvent);
}

bool Session::Connect()
{
	return RegisterConnect();
}

void Session::Disconnect(const WCHAR* cause)
{
	if (_connected.exchange(false) == false)
		return;

	// TEMP
	wcout << "Disconnect : " << cause << endl;

	OnDisconnected(); // 컨텐츠 코드에서 재정의
	GetService()->ReleaseSession(GetSessionRef());

	RegisterDisconnect();
}

HANDLE Session::GetHandle()
{
	return reinterpret_cast<HANDLE>(_socket);
}

void Session::Dispatch(IocpEvent* iocpEvent, int32 numOfBytes)
{
	switch (iocpEvent->eventType)
	{
	case EventType::Connect:
		ProcessConnect();
		break;
	case EventType::Disconnect:
		ProcessDisconnect();
		break;
	case EventType::Recv:
		ProcessRecv(numOfBytes);
		break;
	case EventType::Send:
		ProcessSend(static_cast<SendEvent*>(iocpEvent), numOfBytes);
		break;
	default:
		break;
	}
}

bool Session::RegisterConnect()
{
	if (IsConnected())
		return false;

	if (GetService()->GetServiceType() != ServiceType::Client)
		return false;

	if (SocketUtils::SetReuseAddress(_socket, true) == false)
		return false;

	if (SocketUtils::BindAnyAddress(_socket, 0/*남는거*/) == false)
		return false;

	_connectEvent.Init();
	_connectEvent.owner = shared_from_this(); // ADD_REF

	DWORD numOfBytes = 0;
	SOCKADDR_IN sockAddr = GetService()->GetNetAddress().GetSockAddr();
	if (false == SocketUtils::ConnectEx(_socket, reinterpret_cast<SOCKADDR*>(&sockAddr), sizeof(sockAddr), nullptr, 0, &numOfBytes, &_connectEvent))
	{
		int32 errorCode = ::WSAGetLastError();
		if (errorCode != WSA_IO_PENDING)
		{
			_connectEvent.owner = nullptr; // RELEASE_REF
			return false;
		}
	}

	return true;
}

bool Session::RegisterDisconnect()
{
	_disconnectEvent.Init();
	_disconnectEvent.owner = shared_from_this(); // ADD_REF

	if (false == SocketUtils::DisconnectEx(_socket, &_disconnectEvent, TF_REUSE_SOCKET, 0))
	{
		int32 errorCode = ::WSAGetLastError();
		if (errorCode != WSA_IO_PENDING)
		{
			_disconnectEvent.owner = nullptr; // RELEASE_REF
			return false;
		}
	}

	return true;
}

void Session::RegisterRecv()
{
	if (IsConnected() == false)
		return;

	_recvEvent.Init();
	_recvEvent.owner = shared_from_this(); // ADD_REF

	WSABUF wsaBuf;
	wsaBuf.buf = reinterpret_cast<char*>(_recvBuffer);
	wsaBuf.len = len32(_recvBuffer);

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

void Session::RegisterSend(SendEvent* sendEvent)
{
	if (IsConnected() == false)
		return;

	WSABUF wsaBuf;
	wsaBuf.buf = (char*)sendEvent->buffer.data();
	wsaBuf.len = (ULONG)sendEvent->buffer.size();

	DWORD numOfBytes = 0;
	if (SOCKET_ERROR == ::WSASend(_socket, &wsaBuf, 1, OUT &numOfBytes, 0, sendEvent, nullptr))
	{
		int32 errorCode = ::WSAGetLastError();
		if (errorCode != WSA_IO_PENDING)
		{
			HandleError(errorCode);
			sendEvent->owner = nullptr; // RELEASE_REF
			xdelete(sendEvent);
		}
	}
}

void Session::ProcessConnect()
{
	_connectEvent.owner = nullptr; // RELEASE_REF

	_connected.store(true);

	// 세션 등록
	GetService()->AddSession(GetSessionRef());

	// 컨텐츠 코드에서 재정의
	OnConnected();

	// 수신 등록
	RegisterRecv();
}

void Session::ProcessDisconnect()
{
	_disconnectEvent.owner = nullptr; // RELEASE_REF
}

void Session::ProcessRecv(int32 numOfBytes)
{
	_recvEvent.owner = nullptr; // RELEASE_REF

	if (numOfBytes == 0)
	{
		Disconnect(L"Recv 0");
		return;
	}

	// 컨텐츠 코드에서 재정의
	OnRecv(_recvBuffer, numOfBytes);

	// 수신 등록
	RegisterRecv();
}

void Session::ProcessSend(SendEvent* sendEvent, int32 numOfBytes)
{
	sendEvent->owner = nullptr; // RELEASE_REF
	xdelete(sendEvent);

	if (numOfBytes == 0)
	{
		Disconnect(L"Send 0");
		return;
	}

	// 컨텐츠 코드에서 재정의
	OnSend(numOfBytes);
}

void Session::HandleError(int32 errorCode)
{
	switch (errorCode)
	{
	case WSAECONNRESET:
	case WSAECONNABORTED:
		Disconnect(L"HandleError");
		break;
	default:
		// TODO : Log
		cout << "Handle Error : " << errorCode << endl;
		break;
	}
}
```

Dispatch에서 타입 별 Process- 함수를 호출하면 그 안에서 On-, Register- 함수를 호출하여 작업을 처리하고 이를 콘텐츠 영역에 통지한다. Register- 함수는 예외 상황 처리, event 초기화, 세션 등록, -Ex 함수 호출을 차례로 수행하며, -Ex 함수로는 각각 ConnectEx, DisconnectEx, WSARecv, WSASend를 호출하고 있다. 

<br/> 


# **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버, Rookiss
