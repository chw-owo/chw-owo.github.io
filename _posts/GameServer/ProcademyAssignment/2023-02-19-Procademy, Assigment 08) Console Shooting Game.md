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
#include "Console.h"
#include "Scenes.h"

#include "Title.h"
#include "Load.h"
#include "Game.h"
#include "Clear.h"
#include "Over.h"

#include <iostream>

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

## **2.Scenes**

### **1) Title**

**Title.cpp**
```c++
#include "Title.h"
#include "FileIO.h"
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
	if (GetAsyncKeyState(VK_NUMPAD1)
		|| GetAsyncKeyState(0x31))
	{
		g_Scene = LOAD;
	}
	// Start New Play
	else if (GetAsyncKeyState(VK_NUMPAD2)
		|| GetAsyncKeyState(0x32))
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
	int tmpSize;
	LoadOriginData(sceneFileRoot[TITLE], titleData, tmpSize);
}
```

### **2) Load**

**Load.cpp**
```c++
#include "Load.h"
#include "Render.h"
#include "FileIO.h"
#include "Game.h"

#include <Windows.h>
#include <iostream>

void UpdateLoad()
{
	// logic section
	LoadGameData();
}

void LoadGameData()
{
	int resultSize;

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
	int dataSize = enemyDataCnt * sizeof(EnemyData);
	enemyData = (EnemyData*)(malloc(dataSize));
	memset(enemyData, 0, dataSize);

	for (int i = 0; i < enemyDataCnt; i++)
	{
		LoadTokenedData(gameDataRoot[DATA_ENEMY][i], &enemyData[i],
			resultSize, GET_DATA_ENEMY);

		enemyData[i]._iMax = atoi(gameDataRoot[DATA_ENEMY_NUM][i]);
	}

	InitialEnemy();
	ConvertStageToString();

	g_Scene = GAME;
}

void InitialEnemy()
{
	int iX = 0;
	int iY = 0;
	int enemyMax = 0;

	for (int i = 0; i < enemyDataCnt; i++)
	{
		enemyMax = enemyData[i]._iMax;
		enemyData[i]._enemies
			= (Enemy*)(malloc(sizeof(Enemy) * enemyMax));
	}

	EnemyData* tmp;
	Enemy* enemy;

	for (int i = 0; i < DATA_SIZE; i++)
	{
		if (posData[i] == ' ')
		{
			iX++;
		}
		else if (posData[i] == '\n')
		{
			iY++;
			iX = 0;
		}
		else
		{
			iX++;

			for (int j = 0; j < enemyDataCnt; j++)
			{
				if (posData[i] == *(enemyData[j]._chName))
				{
					tmp = &enemyData[j];
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
#include "Render.h"

#define BULLET_CNT 32

#define ICON_SIZE 8
#define NAME_SIZE 8
#define INFO_SIZE 16

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
	int _iX = dfSCREEN_WIDTH / 2;
	int _iY = (dfSCREEN_HEIGHT / 4) * 3;
	int _iTotalHp = 0;
	int _iHp = 0;
	float _fSpeed = 0;
};

struct Bullet
{
	bool _bDead = false;
	char _chIcon[ICON_SIZE] = { '0', };
	int _iX = -1;
	int _iY = -1;
	int _iAttack = 0;
	float _iSpeed = 0;

};

struct Enemy
{
	bool _bDead = false;
	int _iX = 0;
	int _iY = 0;
	int _iHp = 0;
};

struct EnemyData
{
	char _chName[NAME_SIZE] = { '0', };
	char _chIcon[ICON_SIZE] = { '0', };
	Enemy* _enemies;
	int _iTotalHp = 0;
	int _iMax = 0;
	int _iCnt = 0;
};

extern Player player;
extern Bullet bullets[BULLET_CNT];

extern char posData[DATA_SIZE];
extern EnemyData* enemyData;
extern int enemyDataCnt;

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
#include "FileIO.h"
#include "Scenes.h"
#include <Windows.h>
#include <time.h>

enum GAMESTATE
{
	STATE_PLAY,
	STATE_CLEAR,
	STATE_OVER
};

int gameStageFlag = STATE_PLAY;
char strStage[INFO_SIZE];
Player player;
clock_t moveStart = clock();

// About Enemy ==========================================
char posData[DATA_SIZE];

int enemyDataCnt = 0;
EnemyData* enemyData;

// About bullet ==========================================

Bullet bullets[BULLET_CNT];
int bulletMax = 0;
int bulletCur = 0;
clock_t bulletStart = clock();

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
	clock_t moveEnd = clock();
	float time = (float)(moveEnd - moveStart) / CLOCKS_PER_SEC;

	if (GetAsyncKeyState(VK_ESCAPE))
	{
		g_exit = true;
	}
	else if (GetAsyncKeyState(VK_LEFT))
	{
		if (player._iX > 0 && time > player._fSpeed)
		{
			player._iX--;
			moveStart = clock();
		}
			
	}
	else if (GetAsyncKeyState(VK_RIGHT))
	{
		if (player._iX < dfSCREEN_WIDTH - 3
			&& time > player._fSpeed)
		{
			player._iX++;
			moveStart = clock();
		}		
	}
}


// Update =================================================
void UpdateBullet()
{
	float time = (float)(clock() - bulletStart) / CLOCKS_PER_SEC;

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
		EnemyData* tmp = &enemyData[i];

		for (int j = 0; j < tmp->_iMax; j++)
		{
			Enemy* enemy = &(tmp->_enemies)[j];

			for (int k = 0; k < bulletMax; k++)
			{
				if (enemy->_iX == bullets[k]._iX
					&& enemy->_iY == bullets[k]._iY
					&& bullets[k]._bDead == false
					&& enemy->_bDead == false)
				{
					enemy->_iHp -= bullets[k]._iAttack;
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
		if (bullets[i]._bDead == false)
			rd_SpriteDraw(bullets[i]._iX, bullets[i]._iY, bullets[i]._chIcon[0]);
	}

	Enemy* enemy;
	EnemyData* tmp;

	for (int i = 0; i < enemyDataCnt; i++)
	{
		tmp = &enemyData[i];

		for (int j = 0; j < tmp->_iMax; j++)
		{
			enemy = &(tmp->_enemies)[j];

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
		free(enemyData[i]._enemies);
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

**Clear.cpp**
```c++
#include "Clear.h"
#include "Scenes.h"
#include "Render.h"
#include "FileIO.h"
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
	if (GetAsyncKeyState(VK_RETURN))
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
	int tmpSize;
	char clearDataRoot[ROOT_LEN];
	LoadTokenedData(sceneFileRoot[CLEAR], clearDataRoot,
		tmpSize, GET_ROOT_ONE, g_Stage);

	memset(clearData, ' ', DATA_SIZE);
	LoadOriginData(clearDataRoot, clearData, tmpSize);

}
```

### **5) Over**

**Over.cpp**
```c++
#include "Over.h"
#include "Render.h"
#include "FileIO.h"
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
	int tmpSize;
	LoadOriginData(sceneFileRoot[OVER], overData, tmpSize);
}
```

## **3. Utils**

### **1) For Render**

**Console.h**

```c++
#pragma once
#ifndef __CONSOLE__
#define __CONSOLE__
#define dfSCREEN_WIDTH		81		// 콘솔 가로 80칸 + NULL
#define dfSCREEN_HEIGHT		24		// 콘솔 세로 24칸

void cs_Initial(void);
void cs_MoveCursor(int iPosX, int iPosY);
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

void cs_MoveCursor(int iPosX, int iPosY)
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
#define DATA_SIZE 2048

void rd_BufferFlip(void);
void rd_BufferClear(void);
void rd_SpriteDraw(int iX, int iY, char chSprite);
void rd_DataToBuffer(const char* data);
```

**Render.cpp**

```c++
#include "Render.h"
#include <stdio.h>
#include <memory.h>

char szScreenBuffer[dfSCREEN_HEIGHT][dfSCREEN_WIDTH];

void rd_BufferFlip(void)
{
	int iCnt;
	for (iCnt = 0; iCnt < dfSCREEN_HEIGHT; iCnt++)
	{
		cs_MoveCursor(0, iCnt);
		printf(szScreenBuffer[iCnt]);
	}
}

void rd_BufferClear(void)
{
	int iCnt;
	memset(szScreenBuffer, ' ', dfSCREEN_WIDTH * dfSCREEN_HEIGHT);

	for (iCnt = 0; iCnt < dfSCREEN_HEIGHT; iCnt++)
	{
		szScreenBuffer[iCnt][dfSCREEN_WIDTH - 1] = '\0';
	}
}

void rd_SpriteDraw(int iX, int iY, char chSprite)
{
	if (iX < 0 || iY < 0 || iX >= dfSCREEN_WIDTH - 1 || iY >= dfSCREEN_HEIGHT)
		return;

	szScreenBuffer[iY][iX] = chSprite;
}

void rd_DataToBuffer(const char* data)
{
	int iX = 0;
	int iY = 0;

	for (int i = 0; i < DATA_SIZE; i++)
	{
		if (data[i] == ' ')
		{
			iX++;
		}
		else if (data[i] == '\n')
		{
			iY++;
			iX = 0;
		}
		else
		{
			iX++;
			rd_SpriteDraw(iX, iY, data[i]);
		}
	}
}
```

### **2) For FileIO**

**FileIO.h**

```c++
#pragma once
#include "Game.h"
#include "Scenes.h"

#define ROOT_CNT 16
#define NUM_SIZE 8

enum LOAD_FLAG {

	GET_ROOT_ALL,
	GET_ROOT_ONE,
	GET_GAMEDATA,
	GET_DATA_ENEMY,
	GET_DATA_PLAYER,
	GET_DATA_BULLET
};

void SetStageData(int stage);
void LoadOriginData(const char* fileName, void* output, int& resultSize);
void LoadTokenedData(const char* fileName, void* output, int& resultSize, LOAD_FLAG flag, int idx = 0);

void TokenizeRootAll(char* fileData, const char* sep, int& fileNum, char output[][ROOT_LEN]);
void TokenizeRootOne(char* fileData, const char* sep, int idx, char output[]);
void TokenizeGameData(char* fileData, const char* sep, int& fileNum, char output[][ROOT_CNT][ROOT_LEN]);

void TokenizeEnemyData(char* fileData, const char* sep, EnemyData* output);
void TokenizePlayerData(char* fileData, const char* sep, Player* output);
void TokenizeBulletData(char* fileData, const char* sep, Bullet* output);
```

**FileIO.cpp**

```c++
#include "FileIO.h"
#include <iostream>

void SetStageData(int stage)
{
	g_Stage = stage;

	FILE* pFile;
	char stageData[NUM_SIZE] = { 0, };
	sprintf_s(stageData, NUM_SIZE, "%d", g_Stage);

	fopen_s(&pFile, playData, "w");
	fwrite(stageData, 1, 1, pFile);
	fclose(pFile);
}

void LoadOriginData(const char* fileName, void* output, int& resultSize)
{
	FILE* pFile;

	fopen_s(&pFile, fileName, "r");
	fseek(pFile, 0, SEEK_END);
	resultSize = ftell(pFile);
	rewind(pFile);
	fread((char*)(output), 1, resultSize, pFile);
	fclose(pFile);
}

void LoadTokenedData(const char* fileName, void* output, int& resultSize, LOAD_FLAG flag, int idx)
{
	FILE* pFile;
	char* fileData;

	fopen_s(&pFile, fileName, "r");
	fseek(pFile, 0, SEEK_END);
	resultSize = ftell(pFile);
	rewind(pFile);

	fileData = (char*)(malloc(resultSize));
	memset(fileData, 0, sizeof(char));
	fread(fileData, 1, resultSize, pFile);
	fclose(pFile);

	char sep[] = { "\n" };

	switch (flag)
	{
	case GET_ROOT_ALL:
		TokenizeRootAll(fileData, sep, resultSize,
			(char(*)[ROOT_LEN])(output));
		break;

	case GET_ROOT_ONE:
		TokenizeRootOne(fileData, sep, idx,
			(char*)(output));
		break;

	case GET_GAMEDATA:
		TokenizeGameData(fileData, sep, resultSize,
			(char(*)[ROOT_CNT][ROOT_LEN])(output));
		break;

	case GET_DATA_ENEMY:
		TokenizeEnemyData(fileData, sep, (EnemyData*)(output));
		break;

	case GET_DATA_PLAYER:
		TokenizePlayerData(fileData, sep, (Player*)(output));
		break;

	case GET_DATA_BULLET:
		TokenizeBulletData(fileData, sep, (Bullet*)(output));
		break;


	default:
		break;
	}

	free(fileData);
}

void TokenizeGameData(char* fileData, const char* sep, int& fileNum, char output[][ROOT_CNT][ROOT_LEN])
{
	int idx1 = 0;
	int idx2 = 0;
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

void TokenizeRootOne(char* fileData, const char* sep, int idx, char output[])
{
	int cnt = 0;
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

void TokenizeRootAll(char* fileData, const char* sep, int& fileNum, char output[][ROOT_LEN])
{
	int idx = 0;
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
	int idx = 0;
	char* tok = NULL;
	char* next = NULL;

	tok = strtok_s(fileData, sep, &next);
	strcpy_s(output->_chName, NAME_SIZE, tok);

	tok = strtok_s(NULL, sep, &next);
	strcpy_s(output->_chIcon, ICON_SIZE, tok);

	tok = strtok_s(NULL, sep, &next);
	output->_iTotalHp = atoi(tok);
	output->_iCnt = 0;
}

void TokenizePlayerData(char* fileData, const char* sep, Player* output)
{
	int idx = 0;
	char* tok = NULL;
	char* next = NULL;

	tok = strtok_s(fileData, sep, &next);
	strcpy_s(output->_chIcon, ICON_SIZE, tok);

	tok = strtok_s(NULL, sep, &next);
	output->_iTotalHp = atoi(tok);
	output->_iHp = atoi(tok);

	tok = strtok_s(NULL, sep, &next);
	output->_fSpeed = (float)1 / atoi(tok);
}

void TokenizeBulletData(char* fileData, const char* sep, Bullet* output)
{
	int idx = 0;
	char* tok = NULL;
	char* next = NULL;
	tok = strtok_s(fileData, sep, &next);

	for (int i = 0; i < BULLET_CNT; i++)
		strcpy_s(output[i]._chIcon, ICON_SIZE, tok);

	tok = strtok_s(NULL, sep, &next);
	for (int i = 0; i < BULLET_CNT; i++)
		output[i]._iAttack = atoi(tok);

	tok = strtok_s(NULL, sep, &next);
	for (int i = 0; i < BULLET_CNT; i++)
		output[i]._iSpeed = atof(tok);
}
```

