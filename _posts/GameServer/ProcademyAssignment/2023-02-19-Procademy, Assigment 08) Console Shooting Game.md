---
title: Procademy, Assignment 08) Console Shooting Game
categories: ProcademyAssignment
tags: 
toc: true
toc_sticky: true
---

# **과제**

**-** 윈도우 콘솔 기반의 간단한 슈팅 게임을 구현한다. 

**-** Stage, 위치 데이터, Player, Bullet Enemy 등의 설정값과 같이 가변적인 값들을 파일 외부로 분리한다.

**-** exit 함수, (main 함수를 제외한) 무한 반복 로직을 사용하지 않는다.

**-** 객체지향적인 방식 대신 최대한 절차지향적인 방식으로 구현한다. 

<br/>

# **과제 코드**

## **1. Main**

### **1) Main**

```c++
#include "Extern.h"
#include "Console.h"
#include "Scenes.h"

#include "Title.h"
#include "Load.h"
#include "Game.h"
#include "Clear.h"
#include "Over.h"


int main()
{
	cs_Initial();
	InitialSceneData();
	InitialStageData();

	while (1)
	{	
		switch (g_Scene)
		{
		case TITLE:
			UpdateTitle();
			break;

		case LOAD:
			UpdateLoad();
			break;

		case GAME:
			UpdateGame();
			break;

		case CLEAR:
			UpdateClear();
			break;

		case OVER:
			UpdateOver();
			break;
		}

		if (g_exit == true)
			break;
	}

	return 0;
}
```

### **2) Extern**

**Extern.h**
```c++
#pragma once
#include "Types.h"

#define ROOT_CNT 16
#define ROOT_LEN 128

#define ICON_SIZE 8
#define NUM_SIZE 8
#define NAME_SIZE 8
#define INFO_SIZE 16

#define DATA_SIZE 2048

enum SCENE
{
	TITLE,
	LOAD,
	GAME,
	CLEAR,
	OVER,
	END
};

extern bool g_exit;
extern int32 g_iX;
extern int32 g_Stage;
extern int32 g_Scene;
extern int32 g_finalStage;

extern char rootData[];
extern char playData[];
extern char playFinalData[];
extern char sceneFileRoot[END][ROOT_LEN];

void SetExitTrue(const char func[], const int line);
```


**Extern.cpp**

```c++
#include "Extern.h"
#include "Console.h"
#include <iostream>

bool g_exit = false;
int32 g_iX = 0;
int32 g_Stage = 0;
int32 g_Scene = 0;
int32 g_finalStage;

char rootData[] = ".\\Data\\RootData.txt";
char playData[] = ".\\Data\\PlayData.txt";
char playFinalData[] = ".\\Data\\PlayFinalData.txt";
char sceneFileRoot[END][ROOT_LEN] = { '\0', };

void SetExitTrue(const char func[], const int line)
{
	printf("%s, line %d: \n", func, line);
	g_exit = true;
}
```

### **3) Types**

**Types.h**

```c++
#pragma once

using int8 = __int8;
using int16 = __int16;
using int32 = __int32;
using int64 = __int64;
using uint8 = unsigned __int8;
using uint16 = unsigned __int16;
using uint32 = unsigned __int32;
using uint64 = unsigned __int64;
```

### **4) Scenes**

**Scenes.h**

```c++
#pragma once

void InitialSceneData();
void InitialStageData();
```

**Scenes.cpp**

```c++
#include "Scenes.h"
#include "Extern.h"
#include "UtilFileData.h"
#include <stdlib.h>

void InitialSceneData()
{
	int32 tmpSize;
	LoadTokenedData(rootData, sceneFileRoot, tmpSize, GET_ROOT_ALL);
}

void InitialStageData()
{
	int32 tmpSize;

	char curStage[NUM_SIZE] = { '\0', };
	LoadOriginData(playData, curStage, tmpSize);
	g_Stage = atoi(curStage);

	char finalStageTmp[NUM_SIZE] = { '\0', };
	LoadOriginData(playFinalData, finalStageTmp, tmpSize);
	g_finalStage = atoi(finalStageTmp);
}
```

## **2.Scenes**

### **1) Title**

**Title.h**
```c++
#pragma once

void UpdateTitle();
void GetKeyTitle();
void LoadTitleData();
```

**Title.cpp**
```c++
#include "Title.h"
#include "Extern.h"
#include "UtilFileData.h"
#include "Render.h"
#include <Windows.h>

bool titleInitFlag = false;
char titleData[DATA_SIZE];

void UpdateTitle()
{
	// input section
	GetKeyTitle();

	// logic section
	if (titleInitFlag == false)
	{
		LoadTitleData();
		titleInitFlag = true;
	}	

	// render section
	rd_BufferClear();
	rd_DataToBuffer(titleData);
	rd_BufferFlip();
}

void GetKeyTitle()
{
	// Continue Prev Play
	if (GetAsyncKeyState(VK_NUMPAD1) & 0x0001
		|| GetAsyncKeyState(0x31) & 0x0001)
	{
		g_Scene = LOAD;
	}
	// Start New Play
	else if (GetAsyncKeyState(VK_NUMPAD2) & 0x0001
		|| GetAsyncKeyState(0x32) & 0x0001)
	{
		SetStageData(0);
		g_Scene = LOAD;
	}
	// Exit
	else if (GetAsyncKeyState(VK_ESCAPE))
	{
		g_exit = true;
	}

}


void LoadTitleData()
{
	int32 tmpSize;
	LoadOriginData(sceneFileRoot[TITLE], titleData, tmpSize);
}

```

### **2) Load**

**Load.h**
```c++
#pragma once

// Update
void UpdateLoad();
void GetKeyLoad();

// Load Data
void LoadGameData();

// Process Data
void ConvertStageToString();
void InitialEnemy();
```

**Load.cpp**
```c++
#include "Load.h"
#include "Render.h"
#include "Extern.h"
#include "UtilFileData.h"
#include "Game.h"
#include <Windows.h>
#include <iostream>

void UpdateLoad()
{
	// input section
	//GetKeyLoad();

	// logic section
	LoadGameData();

	// render section
	// rd_BufferClear();
	// rd_DataToBuffer();
	// rd_BufferFlip();
}

void GetKeyLoad()
{
	
}

void LoadGameData()
{
	int32 resultSize;

	// Load Current Stage Data by StageInfo.txt
	char stageDataRoot[ROOT_LEN];
	LoadTokenedData(sceneFileRoot[GAME], stageDataRoot,
		resultSize, GET_ROOT_ONE, g_Stage);

	// Load Data Root by Stage.txt
	char gameDataRoot[DATA_END][ROOT_CNT][ROOT_LEN];
	LoadTokenedData(stageDataRoot, gameDataRoot,
		resultSize, GET_GAMEDATA);
	enemyDataCnt = resultSize;

	// Load Pos Data
	LoadOriginData(gameDataRoot[DATA_POS][0], posData, resultSize);

	// Load Player Data
	LoadTokenedData(gameDataRoot[DATA_PLAYER][0], &player,
		resultSize, GET_DATA_PLAYER);

	// Load Bullet Data
	LoadTokenedData(gameDataRoot[DATA_BULLET][0], bullets,
		resultSize, GET_DATA_BULLET);

	// Load Enemy Data
	int32 dataSize = enemyDataCnt * sizeof(EnemyData);
	enemyData = static_cast<EnemyData*>(malloc(dataSize));
	memset(enemyData, 0, dataSize);

	for (int i = 0; i < enemyDataCnt; i++)
	{
		LoadTokenedData(gameDataRoot[DATA_ENEMY][i], enemyData + i,
			resultSize, GET_DATA_ENEMY);

		(enemyData + i)->_iMax = atoi(gameDataRoot[DATA_ENEMY_NUM][i]);
	}
	
	InitialEnemy();
	ConvertStageToString();
	
	g_Scene = GAME;
}

void InitialEnemy()
{
	int32 iX = 0;
	int32 iY = 0;
	int enemyMax = 0;

	for (int i = 0; i < enemyDataCnt; i++)
	{
		enemyMax = (enemyData + i)->_iMax;
		(enemyData + i)->_enemies
			= static_cast<Enemy*>(malloc(sizeof(Enemy) * enemyMax));
	}

	EnemyData* tmp;
	Enemy* enemy;

	for (int i = 0; i < DATA_SIZE; i++)
	{
		if (*(posData + i) == ' ')
		{
			iX++;
		}
		else if (*(posData + i) == '\n')
		{
			iY++;
			iX = 0;
		}
		else
		{
			iX++;

			for (int j = 0; j < enemyDataCnt; j++)
			{
				if (*(posData + i) == *(enemyData + j)->_chName)
				{
					tmp = enemyData + j;
					enemy = (tmp->_enemies) + (tmp->_iCnt);

					enemy->_bDead = false;
					enemy->_iX = iX;
					enemy->_iY = iY;
					enemy->_iHp = tmp->_iTotalHp;

					tmp->_iCnt++;
				}
			}
		}
	}
}

void ConvertStageToString()
{
	sprintf_s(strStage, INFO_SIZE, "Stage: %d", g_Stage + 1);
}
```

### **3) Game**

**Game.h**
```c++
#pragma once
#include "Extern.h"
#include "Render.h"

#define BULLET_CNT 32

enum GAMEDATA
{
	DATA_POS,
	DATA_PLAYER,
	DATA_BULLET,
	DATA_ENEMY,
	DATA_ENEMY_NUM,
	DATA_END
};

struct Player
{
	bool _bDead = false;
	char _chIcon[ICON_SIZE] = { '0', };
	int32 _iX = dfSCREEN_WIDTH / 2;
	int32 _iY = (dfSCREEN_HEIGHT / 4) * 3;
	int32 _iTotalHp = 0;
	int32 _iHp = 0;
};

struct Bullet
{
	bool _bDead = false;
	char _chIcon[ICON_SIZE] = { '0', };
	int32 _iX = -1;
	int32 _iY = -1;
	int32 _iAttack = 0;
	float _iSpeed = 0;

};

struct Enemy
{
	bool _bDead = false;
	int32 _iX = 0;
	int32 _iY = 0;
	int32 _iHp = 0;
};

struct EnemyData
{
	char _chName[NAME_SIZE] = { '0', };
	char _chIcon[ICON_SIZE] = { '0', };
	Enemy* _enemies;
	int32 _iTotalHp = 0;
	int32 _iMax = 0;
	int32 _iCnt = 0;
};

extern Player player;
extern Bullet bullets[BULLET_CNT];

extern char posData[DATA_SIZE];
extern EnemyData* enemyData;
extern int32 enemyDataCnt;

extern char strStage[INFO_SIZE];

// Main
void UpdateGame();

// Get Input
void GetKeyGame();

// Update
void UpdateBullet();
void UpdateEnemy();
void UpdatePlayer();

// Render
void GameDataToBuffer();

// Scene Change
void FreeHeapData();
void GameClear();
void GameOver();

```

**Game.cpp**
```c++
#include "Game.h"
#include "UtilFileData.h"
#include <Windows.h>
#include <time.h>

enum GAMESTATE
{
	STATE_PLAY,
	STATE_CLEAR,
	STATE_OVER
};

int32 gameStageFlag = STATE_PLAY;
char strStage[INFO_SIZE];
Player player;

// About Enemy ==========================================
char posData[DATA_SIZE];

int32 enemyDataCnt = 0;
EnemyData* enemyData;

// About bullet ==========================================

Bullet bullets[BULLET_CNT];
int32 bulletMax = 0;
int32 bulletCur = 0;
clock_t bulletStart = clock();
clock_t bulletEnd = clock();

// Function ==============================================

void UpdateGame()
{
	switch (gameStageFlag)
	{
	case STATE_PLAY:

		// input section
		GetKeyGame();

		// logic section
		UpdatePlayer();
		UpdateBullet();
		UpdateEnemy();

		// render section
		rd_BufferClear();
		GameDataToBuffer();
		rd_BufferFlip();

		break;

	case STATE_CLEAR:
		GameClear();
		break;

	case STATE_OVER:
		GameOver();
		break;
	}
}
 
// Get Input =================================================

void GetKeyGame()
{
	if (GetAsyncKeyState(VK_ESCAPE))
	{
		g_exit = true;
	}
	else if (GetAsyncKeyState(VK_LEFT) & 0x0001)
	{
		if (player._iX > 0)
			player._iX--;
	}
	else if (GetAsyncKeyState(VK_RIGHT) & 0x0001)
	{
		if (player._iX < dfSCREEN_WIDTH - 3)
			player._iX++;
	}
}

// Update =================================================
void UpdateBullet()
{
	bulletEnd = clock();
	float time = (float)(bulletEnd - bulletStart) / CLOCKS_PER_SEC;

	if (time < bullets[bulletCur]._iSpeed)
		return;

	if (bulletMax < BULLET_CNT)
		bulletMax++;

	bullets[bulletCur]._iX = player._iX;
	bullets[bulletCur]._iY = player._iY;
	bullets[bulletCur]._bDead = false;

	for (int i = 0; i < bulletMax; i++)
	{
		bullets[i]._iY--;
	}

	bulletCur++;

	if (bulletCur >= BULLET_CNT)
		bulletCur = 0;

	bulletStart = clock();
}

void UpdateEnemy()
{
	int enemyCnt = 0; 

	for (int i = 0; i < enemyDataCnt; i++)
	{
		EnemyData* tmp = enemyData + i;

		for (int j = 0; j < tmp -> _iMax; j++)
		{
			Enemy* enemy = (tmp -> _enemies) + j;

			for (int k = 0; k < bulletMax; k++)
			{
				if (bullets[k]._bDead == false
					&& enemy -> _bDead == false
					&& enemy -> _iX == bullets[k]._iX
					&& enemy -> _iY == bullets[k]._iY)
				{
					enemy -> _iHp -= bullets[k]._iAttack;
					
					bullets[k]._bDead = true;
				}
			}

			if (enemy->_iHp <= 0)
				enemy->_bDead = true;
			else
				enemyCnt++;
		}
	}

	if (enemyCnt <= 0)
	{
		gameStageFlag = STATE_CLEAR;
		return;
	}

}

void UpdatePlayer()
{
	if (player._iHp <= 0)
	{
		gameStageFlag = STATE_OVER;
		return;
	}
}

// Render =================================================

void GameDataToBuffer()
{
	for (int i = 0; i < bulletMax; i++)
	{
		if(bullets[i]._bDead == false)
			rd_SpriteDraw(bullets[i]._iX, bullets[i]._iY, bullets[i]._chIcon[0]);
	}

	Enemy* enemy;
	EnemyData* tmp;

	for (int i = 0; i < enemyDataCnt; i++)
	{
		tmp = enemyData + i;

		for (int j = 0; j < tmp -> _iMax; j++)
		{
			enemy = tmp->_enemies + j;

			if (enemy->_bDead == false)
				rd_SpriteDraw(enemy->_iX, enemy->_iY, *tmp->_chIcon);
		}
	}

	rd_SpriteDraw(player._iX, player._iY, player._chIcon[0]);

	for (int i = 0; i < strlen(strStage); i++)
	{
		rd_SpriteDraw(i, 0, strStage[i]);
	}	
}

// Change Scene =============================================

void FreeHeapData()
{
	for (int i = 0; i < enemyDataCnt; i++)
	{
		free((enemyData + i)->_enemies);
	}
	free(enemyData);
}

void GameClear()
{
	FreeHeapData();
	gameStageFlag = STATE_PLAY;
	g_Scene = CLEAR;
}

void GameOver()
{
	FreeHeapData();
	gameStageFlag = STATE_PLAY;
	g_Scene = OVER;
}
```

### **4) Clear**

**Clear.h**
```c++
#pragma once

void UpdateClear();
void GetKeyClear();
void InitialClear();
```

**Clear.cpp**
```c++
#include "Clear.h"
#include "Render.h"
#include "Extern.h"
#include "UtilFileData.h"
#include <Windows.h>
#include <iostream>

bool clearInitFlag = false;
char clearData[DATA_SIZE];

void UpdateClear()
{
	// input section
	GetKeyClear();

	// logic section
	if (clearInitFlag == false)
	{
		InitialClear();
		clearInitFlag = true;
	}
		
	// render section
	rd_BufferClear();
	rd_DataToBuffer(clearData);
	rd_BufferFlip();
}

void GetKeyClear()
{
	if (GetAsyncKeyState(VK_RETURN) & 0x0001)
	{
		if (g_Stage >= g_finalStage)
		{
			clearInitFlag = false;
			g_Scene = TITLE;
		}
		else
		{
			clearInitFlag = false;
			SetStageData(g_Stage + 1);
			g_Scene = LOAD;
		}
	}
	else if (GetAsyncKeyState(VK_ESCAPE))
	{
		clearInitFlag = false;
		g_exit = true;
	}
}

void InitialClear()
{
	int32 tmpSize;
	char clearDataRoot[ROOT_LEN];
	LoadTokenedData(sceneFileRoot[CLEAR], clearDataRoot,
		tmpSize, GET_ROOT_ONE, g_Stage);

	memset(clearData, ' ', DATA_SIZE);
	LoadOriginData(clearDataRoot, clearData, tmpSize);

}
```

### **5) Over**

**Over.h**
```c++
#pragma once

void UpdateOver();
void GetKeyOver();
void LoadOverData();
```

**Over.cpp**
```c++
#include "Over.h"
#include "Render.h"
#include "Extern.h"
#include "UtilFileData.h"
#include <Windows.h>

bool overInitFlag;
char overData[DATA_SIZE];

void UpdateOver()
{
	// input section
	GetKeyOver();

	// logic section
	if (overInitFlag == false)
	{
		LoadOverData();
		overInitFlag = true;
	}

	// render section
	rd_BufferClear();
	rd_DataToBuffer(overData);
	rd_BufferFlip();
}

void GetKeyOver()
{
	if (GetAsyncKeyState(VK_RETURN))
	{
		g_Scene = GAME;
	}
	else if (GetAsyncKeyState(VK_ESCAPE))
	{
		g_exit = true;
	}
}

void LoadOverData()
{
	int32 tmpSize;
	LoadOriginData(sceneFileRoot[OVER], overData, tmpSize);
}
```

## **3. Utils**

### **1) For Render**

**Console.h**

```c++
#pragma once

//////////////////////////////////////////////////////////////
// Windows 의 콘솔 화면에서 커서 제어.							//
//////////////////////////////////////////////////////////////

#ifndef __CONSOLE__
#define __CONSOLE__
#define dfSCREEN_WIDTH		81		// 콘솔 가로 80칸 + NULL
#define dfSCREEN_HEIGHT		24		// 콘솔 세로 24칸

#include "Types.h"

void cs_Initial(void);
void cs_MoveCursor(const int32& iPosX, const int32& iPosY);
void cs_ClearScreen(void);


#endif
```

**Console.cpp**

```c++
#include "Console.h"
#include <windows.h>

HANDLE  hConsole;

void cs_Initial(void)
{
	CONSOLE_CURSOR_INFO stConsoleCursor;

	stConsoleCursor.bVisible = FALSE;
	stConsoleCursor.dwSize = 1;

	hConsole = GetStdHandle(STD_OUTPUT_HANDLE);
	SetConsoleCursorInfo(hConsole, &stConsoleCursor);
}

void cs_MoveCursor(const int32& iPosX, const int32& iPosY)
{
	COORD stCoord;
	stCoord.X = iPosX;
	stCoord.Y = iPosY;

	SetConsoleCursorPosition(hConsole, stCoord);
}


void cs_ClearScreen(void)
{
	DWORD dw;
	FillConsoleOutputCharacter(GetStdHandle(STD_OUTPUT_HANDLE), ' ', 100 * 100, { 0, 0 }, &dw);
}
```

**Render.h**

```c++
#pragma once
#include "Console.h"
#include "Types.h"
#include "Extern.h"

void rd_BufferFlip(void);
void rd_BufferClear(void);
void rd_SpriteDraw(const int32& iX, const int32& iY, const char& chSprite);
void rd_DataToBuffer(const char* data);
```

**Render.cpp**

```c++
#include "Render.h"
#include <stdio.h>
#include <memory.h>

char szScreenBuffer[dfSCREEN_HEIGHT][dfSCREEN_WIDTH];

// 버퍼의 내용을 화면으로 찍어주는 함수.
// 적군,아군,총알 등을 szScreenBuffer 에 넣어주고, 
// 한 프레임이 끝나는 마지막에 본 함수를 호출하여 버퍼 -> 화면 으로 그린다.
void rd_BufferFlip(void)
{
	int32 iCnt;
	for (iCnt = 0; iCnt < dfSCREEN_HEIGHT; iCnt++)
	{
		cs_MoveCursor(0, iCnt);
		printf(szScreenBuffer[iCnt]);
	}
}

// 화면 버퍼를 지워주는 함수
// 매 프레임 그림을 그리기 직전에 버퍼를 지워 준다. 
void rd_BufferClear(void)
{
	int32 iCnt;
	memset(szScreenBuffer, ' ', dfSCREEN_WIDTH * dfSCREEN_HEIGHT);

	for (iCnt = 0; iCnt < dfSCREEN_HEIGHT; iCnt++)
	{
		szScreenBuffer[iCnt][dfSCREEN_WIDTH - 1] = '\0';
	}
}

// 버퍼의 특정 위치에 원하는 문자를 출력.
// 입력 받은 X,Y 좌표에 아스키코드 하나를 출력한다. (버퍼에 그림)
void rd_SpriteDraw(const int32& iX, const int32& iY, const char& chSprite)
{
	if (iX < 0 || iY < 0 || iX >= dfSCREEN_WIDTH - 1 || iY >= dfSCREEN_HEIGHT)
		return;

	szScreenBuffer[iY][iX] = chSprite;
}

void rd_DataToBuffer(const char* data)
{
	int32 iX = 0;
	int32 iY = 0;

	for (int i = 0; i < DATA_SIZE; i++)
	{
		if (*(data + i) == ' ')
		{
			iX++;
		}
		else if (*(data + i) == '\n')
		{
			iY++;
			iX = 0;
		}
		else
		{
			iX++;
			rd_SpriteDraw(iX, iY, *(data + i));
		}
	}
}
```



### **2) For FileIO**

**FileFunction.h**

```c++
#pragma once
#include "Types.h"
#include <stdio.h>

//pFile IO
void OpenFile(const char* filename, FILE** pFile, const char* mode);
void ReadFile(FILE** pFile, const int32& size, char* data);
void WriteFile(FILE** pFile, const int32& size, char* data);
void CloseFile(FILE** pFile);
int32 GetFileSize(FILE** pFile);
```

**FileFunction.cpp**

```c++
#include "FileFunction.h"
#include "Extern.h"

// File IO
void OpenFile(const char* filename, FILE** pFile, const char* mode)
{
	errno_t ret = fopen_s(pFile, filename, mode);

	if (ret != 0)
	{
		printf("Fail to open %s\n", filename);
		printf("errno_t : %d\n", ret);
		SetExitTrue(__FUNCTION__, __LINE__);
	}
}

void CloseFile(FILE** pFile)
{
	errno_t ret = fclose(*pFile);
	if (ret != 0)
	{
		printf("Fail to close pFile : %d\n", ret);
		SetExitTrue(__FUNCTION__, __LINE__);
	}

}

void ReadFile(FILE** pFile, const int32& size, char* data)
{
	fread(data, 1, size, *pFile);
	errno_t ret = ferror(*pFile);
	if (ret != 0)
	{
		printf("Fail to read data : %d\n", ret);
		SetExitTrue(__FUNCTION__, __LINE__);
	}

}

void WriteFile(FILE** pFile, const int32& size, char* data)
{
	fwrite(data, size, 1, *pFile);
	errno_t ret = ferror(*pFile);

	if (ret != 0)
	{
		printf("Fail to write data : %d\n", ret);
		SetExitTrue(__FUNCTION__, __LINE__);
	}

}

int32 GetFileSize(FILE** pFile)
{
	fseek(*pFile, 0, SEEK_END);
	int32 size = ftell(*pFile);
	rewind(*pFile);

	if (size == -1)
	{
		printf("Fail to get file size\n");
		SetExitTrue(__FUNCTION__, __LINE__);
	}

	return size;
}
```

**UtilFileData.h**

```c++
#pragma once
#include "Extern.h"
#include "Types.h"
#include "Game.h"

enum LOAD_FLAG {

	GET_ROOT_ALL,
	GET_ROOT_ONE,
	GET_GAMEDATA,
	GET_DATA_ENEMY,
	GET_DATA_PLAYER,
	GET_DATA_BULLET
};

void SetStageData(const int32 &stage);
void LoadOriginData(const char* fileName, void* output, int32& resultSize);
void LoadTokenedData(const char* fileName, void* output, int32& resultSize, const LOAD_FLAG& flag, const int32& idx = 0);

void TokenizeRootAll(char* fileData, const char* sep, int32& fileNum, char output[][ROOT_LEN]);
void TokenizeRootOne(char* fileData, const char* sep, const int32& idx, char output[]);
void TokenizeGameData(char* fileData, const char* sep, int32& fileNum, char output[][ROOT_CNT][ROOT_LEN]);

void TokenizeEnemyData(char* fileData, const char* sep, EnemyData* output);
void TokenizePlayerData(char* fileData, const char* sep, Player* output);
void TokenizeBulletData(char* fileData, const char* sep, Bullet* output);
```

**UtilFileData.cpp**

```c++
#include "UtilFileData.h"
#include "FileFunction.h"
#include <iostream>

void SetStageData(const int32& stage)
{
	g_Stage = stage;

	FILE* pFile;
	char data[NUM_SIZE] = { 0, };
	sprintf_s(data, NUM_SIZE, "%d", g_Stage);
	OpenFile(playData, &pFile, "w");
	WriteFile(&pFile, 1, data);
	CloseFile(&pFile);
}

void LoadOriginData(const char* fileName, void* output, int32& resultSize)
{
	FILE* pFile;
	OpenFile(fileName, &pFile, "r");
	resultSize = GetFileSize(&pFile);
	ReadFile(&pFile, resultSize, static_cast<char*>(output));
	CloseFile(&pFile);
	return;
}

void LoadTokenedData(const char* fileName, void* output, int32& resultSize, 
						const LOAD_FLAG& flag, const int32& idx)
{
	FILE* pFile;
	char* fileData;

	OpenFile(fileName, &pFile, "r");
	resultSize = GetFileSize(&pFile);
	fileData = static_cast<char*>(calloc(resultSize, sizeof(char)));
	ReadFile(&pFile, resultSize, fileData);
	CloseFile(&pFile);

	char sep[] = { "\n" };

	switch (flag)
	{
		case GET_ROOT_ALL:
			TokenizeRootAll(fileData, sep, resultSize,
								static_cast<char(*)[ROOT_LEN]>(output));
			break;

		case GET_ROOT_ONE:
			TokenizeRootOne(fileData, sep, idx,
								static_cast<char*>(output));
			break;

		case GET_GAMEDATA:
			TokenizeGameData(fileData, sep, resultSize,
				static_cast<char(*)[ROOT_CNT][ROOT_LEN]>(output));
			break;

		case GET_DATA_ENEMY:
			TokenizeEnemyData(fileData, sep, static_cast<EnemyData*>(output));
			break;

		case GET_DATA_PLAYER:
			TokenizePlayerData(fileData, sep, static_cast<Player*>(output));
			break;

		case GET_DATA_BULLET:
			TokenizeBulletData(fileData, sep, static_cast<Bullet*>(output));
			break;
		

		default:
			break;
	}

	free(fileData);
}

void TokenizeGameData(char* fileData, const char* sep, int32& fileNum, char output[][ROOT_CNT][ROOT_LEN])
{
	int32 idx1 = 0;
	int32 idx2 = 0;
	char* tok = NULL;
	char* next = NULL;

	tok = strtok_s(fileData, sep, &next);
	while (tok != NULL)
	{
		if (strcmp(tok, "POSITION") == 0)
		{
			idx1 = DATA_POS;
			idx2 = 0;
		}
		else if (strcmp(tok, "PLAYER") == 0)
		{
			idx1 = DATA_PLAYER;
			idx2 = 0;
		}
		else if (strcmp(tok, "BULLET") == 0)
		{
			idx1 = DATA_BULLET;
			idx2 = 0;
		}
		else if (strcmp(tok, "ENEMY") == 0)
		{
			idx1 = DATA_ENEMY;
			idx2 = 0;
		}
		else if (strcmp(tok, "ENEMYNUM") == 0)
		{
			idx1 = DATA_ENEMY_NUM;
			idx2 = 0;
		}
		else
		{
			strcpy_s(output[idx1][idx2], ROOT_LEN, tok);
			idx2++;
		}
		
		tok = strtok_s(NULL, sep, &next);
	}

	fileNum = idx2;
}

void TokenizeRootOne(char* fileData, const char* sep, const int32& idx, char output[])
{
	int32 cnt = 0;
	char* tok = NULL;
	char* next = NULL;
	
	tok = strtok_s(fileData, sep, &next);
	while (cnt < idx && tok != NULL)
	{
		tok = strtok_s(NULL, sep, &next);
		cnt++;
	}
	strcpy_s(output, ROOT_LEN, tok);
}

void TokenizeRootAll(char* fileData, const char* sep, int32& fileNum, char output[][ROOT_LEN])
{
	int32 idx = 0;
	char* tok = NULL;
	char* next = NULL;

	tok = strtok_s(fileData, sep, &next);
	while (tok != NULL)
	{
		strcpy_s(output[idx], ROOT_LEN, tok);
		tok = strtok_s(NULL, sep, &next);
		idx++;
	}
	fileNum = idx;
}


void TokenizeEnemyData(char* fileData, const char* sep, EnemyData* output)
{
	int32 idx = 0;
	char* tok = NULL;
	char* next = NULL;

	tok = strtok_s(fileData, sep, &next);
	strcpy_s(output->_chName, NAME_SIZE, tok);

	tok = strtok_s(NULL, sep, &next);
	strcpy_s(output->_chIcon, ICON_SIZE, tok);

	tok = strtok_s(NULL, sep, &next);
	output-> _iTotalHp = atoi(tok);
	output-> _iCnt = 0;
}

void TokenizePlayerData(char* fileData, const char* sep, Player* output)
{
	int32 idx = 0;
	char* tok = NULL;
	char* next = NULL;

	tok = strtok_s(fileData, sep, &next);
	strcpy_s(output->_chIcon, ICON_SIZE, tok);

	tok = strtok_s(NULL, sep, &next);
	output->_iTotalHp = atoi(tok);
	output->_iHp = atoi(tok);
}

void TokenizeBulletData(char* fileData, const char* sep, Bullet* output)
{
	int32 idx = 0;
	char* tok = NULL;
	char* next = NULL;
	tok = strtok_s(fileData, sep, &next);

	for(int i = 0; i < BULLET_CNT; i++)
		strcpy_s((output + i)->_chIcon, ICON_SIZE, tok);

	tok = strtok_s(NULL, sep, &next);
	for (int i = 0; i < BULLET_CNT; i++)
		(output + i)->_iAttack = atoi(tok);

	tok = strtok_s(NULL, sep, &next);
	for (int i = 0; i < BULLET_CNT; i++)
		(output + i)->_iSpeed = atof(tok);
}
```

