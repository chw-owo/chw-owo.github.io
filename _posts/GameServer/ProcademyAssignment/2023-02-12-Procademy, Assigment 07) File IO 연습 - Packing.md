---
title: Procademy, Assignment 07) File IO 연습 - Packing
categories: ProcademyAssignment
tags: 
toc: true
toc_sticky: true
---

# **과제 설명**

zip 처럼 파일을 패킹할 수 있는 프로그램을 만든다. 이 과정을 통해 파일 입출력에 익숙해진다. 

**1. 프로그램 구조**

**1)** 패킹 or 언패킹 선택 

**2)**  패킹 선택 > 패킹할 파일 개수 입력받기 > 각각의 파일이름 입력 받기 > 패킹!

**3)** 언패킹 선택 > 패킹파일 입력받기 > 해당파일의 헤더를 분석하여 파일 분리 파일로 저장!

<br/>

**2. 파일패킹 과정**

**1)** 합치고자 하는 파일의 존재유무 확인.

**2)** 합치고자 하는 파일들의 이름,사이즈 얻기

**3)** 패킹파일을 열어서 stPACK_HEADER 헤더 저장

**4)** 패킹파일에 파일헤더 stPACK_FILEINFO 개수만큼 저장

**5)** 합칠 파일들을 하나씩 열면서 내용 패킹 파일로 저장

<br/>

**3. 파일언패킹 과정**

**1)** 패킹파일 오픈

**2)** 패킹파일 stPACK_HEADER 해더 읽기.  Type 확인 패킹파일 확인.

**3)** 파일개수 확인

**4)** 파일 개수만큼 stPACK_FILEINFO 헤더 읽기

**5)** 파일 개수만큼 돌면서 stPACK_FILEINFO[N] 의 정보로 파일생성

**6)** 사이즈만큼 읽어서 저장

<br/>

# **과제 코드**

처음에는 파일을 pack 구조체에 쓰는 것, pack 구조체의 내용을 수정하는 것, pack 구조체의 내용을 파일에 쓰는 것을 각각 함수로 분리해서 구현을 했다. 그런데 진행하다보니 그렇게 여러번 복사가 이루어지게 하는 것보다 한 함수에서 읽고 수정하고 쓰는 과정을 합쳐서 진행하는 게 더 효율적일 것 같단 생각이 들었다. 그래서 pack/ unpack/ add file to pack/ unpack selected file /update file 이렇게 다섯가지 함수로만 나누는 걸로 수정하고 나눴던 함수들을 한 함수에 합치는 작업을 하다가... 힘들어서 돌아갈 수 있게만 만들어놓고 중간에 멈췄다... 그래서 코드가 지저분하고 비효율적인 상태. 

현재는 pack에 복사하여 수정한 뒤 파일에 쓰는 방식이지만 추후에는 pack에 복사하는 대신 바로 수정하여 인덱스로 접근해 파일에 쓰는 방식으로 수정할 예정이다. 

<br/>

**main.cpp**
```c++
#include "FileIO.h"
#include "PackingMenu.h"

int _tmain(int argc, _TCHAR* argv[])
{
	while (1)
	{
		PrintMenu();
		SelectMenu();
	}
}
```

**FileIO.h**
```c++
#pragma once
#include <iostream>
#include <TCHAR.h>
#include <windows.h>

//File IO
void OpenFile(const TCHAR* filename, FILE** file, const TCHAR* mode);
void ReadFile(FILE** file, const DWORD& size, char* data);
void WriteFile(FILE** file, const DWORD& size, char* data);
void CloseFile(FILE** file);
DWORD GetFileSize(FILE** file);
```

**FileIO.cpp**
```c++
#include "FileIO.h"
#include "Packing.h"

// File IO
void OpenFile(const TCHAR* filename, FILE** file, const TCHAR* mode)
{
	errno_t ret = _tfopen_s(&(*file), filename, mode);
	if (ret != 0)
		_tprintf(_T("Fail to open %s : %d\n"), filename, ret);
}

void CloseFile(FILE** file)
{
	errno_t ret = fclose(*file);
	if (ret != 0)
		_tprintf(_T("Fail to close file : %d\n"), ret);
}

void ReadFile(FILE** file, const DWORD& size, char* data)
{
	fread(data, size, 1, *file);
	errno_t ret = ferror(*file);

	if (ret != 0)
		_tprintf(_T("Fail to read data : %d\n"), ret);
}

void WriteFile(FILE** file, const DWORD& size, char* data)
{
	fwrite(data, size, 1, *file);
	errno_t ret = ferror(*file);

	if (ret != 0)
		_tprintf(_T("Fail to write data : %d\n"), ret);
}

DWORD GetFileSize(FILE** file)
{
	fseek(*file, 0, SEEK_END);
	LONG size = ftell(*file);
	fseek(*file, 0, SEEK_SET);

	if (size == -1)
		_tprintf(_T("Fail to get file size\n"));

	return size;
}
```

**PackingMenu.h**
```c++
#pragma once
#include "Packing.h"

enum MENU
{
	PACK = 1,
	UNPACK,
	SHOW_FILES,
	ADD_PACK,
	SELECT_UNPACK,
	SELECT_UPDATE,
	CLEAR_CONSOLE
};

//Menu
void SelectMenu();
void PrintMenu();

void Menu_Pack();
void Menu_Unpack();
void Menu_ShowFiles();
void Menu_AddPack();
void Menu_UnpackFiles();
void Menu_UpdateFile();
```

**PackingMenu.cpp**
```c++
#include "PackingMenu.h"

void PrintMenu()
{
	_tprintf(_T(
		"1. Pack\n"
		"2. Unpack\n"
		"3. Show All Files in Packfile\n"
		"4. Add Files to Packing File\n"
		"5. Select Files to Unpack\n"
		"6. Select File to Update\n"
		"7. Clear Console Window\n\n"

		"Choose number and press enter\n")
	);
}

void SelectMenu()
{
	TCHAR input[MENU_INPUT] = { '\0', };
	_fgetts(input, MENU_INPUT, stdin);
	DWORD select = _ttoi(input);

	switch (select)
	{
	case PACK:
		Menu_Pack();
		break;

	case UNPACK:
		Menu_Unpack();
		break;

	case SHOW_FILES:
		Menu_ShowFiles();
		break;

	case ADD_PACK:
		Menu_AddPack();
		break;

	case SELECT_UNPACK:
		Menu_UnpackFiles();
		break;

	case SELECT_UPDATE:
		Menu_UpdateFile();
		break;

	case CLEAR_CONSOLE:
		system("cls");
		break;

	default:
		_tprintf(_T("Wrong input. Choose Again\n"));
	}
}

// Menu

void Menu_Pack()
{
	// Get PackFile Name
	TCHAR packName[FILE_NAME] = { '\0', };
	_tprintf(_T("Enter pack file name. (max length %u)\n"), FILE_NAME);
	_fgetts(packName, FILE_NAME, stdin);
 
	// Get Files' Name to Pack
	TCHAR fileNames[INPUT] = { ' ', };
	_tprintf(_T("Enter files' name with extension to pack." 
		" (max count %d, max length %d)\n"), FILE_CNT, INPUT);
	_fgetts(fileNames, INPUT, stdin);

	Pack(packName, fileNames);

}

void Menu_Unpack()
{
	// Get PackFile Name
	TCHAR packName[FILE_NAME] = { '\0', };
	_tprintf(_T("Enter pack file name to unpack.\n"));
	_fgetts(packName, FILE_NAME, stdin);

	Unpack(packName);
}

void Menu_ShowFiles()
{
	// Get PackFile Name
	TCHAR packName[FILE_NAME] = { '\0', };
	_tprintf(_T("Enter pack file name to add file.\n"));
	_fgetts(packName, FILE_NAME, stdin);

	//ReadPackFile(packName, pack);
	ShowFiles(packName);

	_tprintf(_T("Show All Files Complete!\n\n"));
}

void Menu_AddPack()
{
	// Get PackFile Name
	TCHAR packName[FILE_NAME] = { '\0', };
	_tprintf(_T("Enter pack file name to add file.\n"));
	_fgetts(packName, FILE_NAME, stdin);

	// Get Files' Name to Pack
	TCHAR fileNames[INPUT] = { ' ', };
	_tprintf(_T("Enter files name with extension to add."
		" (max count %d, max length %d)\n"), FILE_CNT, INPUT);
	_fgetts(fileNames, INPUT, stdin);

	// Add Files To Pack
	AddPack(packName, fileNames);
}

void Menu_UnpackFiles()
{
	// Get PackFile Name
	TCHAR packName[FILE_NAME] = { '\0', };
	_tprintf(_T("Enter pack file name to unpack.\n"));
	_fgetts(packName, FILE_NAME, stdin);

	// Get Files' Name to Pack
	TCHAR fileNames[INPUT] = { ' ', };
	_tprintf(_T("Enter files name with extension to add."
		" (max count %d, max length %d)\n"), FILE_CNT, INPUT);
	_fgetts(fileNames, INPUT, stdin);

	// Write Data to Files
	UnpackFiles(packName, fileNames);
}

void Menu_UpdateFile()
{
	// Get Packfile Name
	TCHAR packName[FILE_NAME] = { '\0', };
	_tprintf(_T("Enter pack file name to unpack.\n"));
	_fgetts(packName, FILE_NAME, stdin);

	// Get File Name to Update
	TCHAR fileName[FILE_NAME] = { ' ', };
	_tprintf(_T("Enter file name with extension to update."
		" (max length %d)\n"), FILE_NAME);
	_fgetts(fileName, FILE_NAME, stdin);

	// Update Data
	UpdateFile(packName, fileName);

}
```

**Packing.h**
```c++
#pragma once
#include "FileIO.h"

#define MENU_INPUT 4
#define FILE_CNT 32
#define FILE_NAME 32
#define INPUT 512
#define TYPE 0x99886655
typedef struct _PACKINFOHEADER
{
	DWORD64 type = 0x99886655;
	DWORD fileNum = 0;

}PACKINFOHEADER;

typedef struct _FILEINFOHEADER
{
	DWORD fileSize[FILE_CNT] = { 0, };
	DWORD offset[FILE_CNT] = { 0, };
	TCHAR fileName[FILE_CNT][FILE_NAME];

}FILEINFOHEADER;

typedef struct _PACKING
{
	PACKINFOHEADER pi;
	FILEINFOHEADER fi;
	char* data;

}PACKING;

void Pack(TCHAR* packName, TCHAR* inputFilesName);
void Unpack(TCHAR* packName);
void ShowFiles(TCHAR* packName);
void AddPack(TCHAR* packName, TCHAR* inputFilesName);
void UnpackFiles(TCHAR* packName, TCHAR* inputFilesName);
void UpdateFile(TCHAR* packName, TCHAR* fileName);

//For Common ================================
void RemoveChar(TCHAR* fileName);
void SetPack(PACKING& pack);

template <typename T>
void ChangeDataToChar(char* output, const DWORD& idx, const T& input);
template <typename T>
void ChangeDataArrToChar(char* output, const T& input, const DWORD& idx, const DWORD& cnt);
template <typename T>
void ChangeCharToData(const char* input, const DWORD& idx, T& output);
template <typename T>
void ChangeCharToDataArr(const char* input, T& output, const DWORD& idx, const DWORD& cnt);
```

**Packing.cpp**
```c++
#include "Packing.h"
#include "FileIO.h"

void Pack(TCHAR* packName, TCHAR* inputFilesName)
{
	PACKING pack;
	RemoveChar(packName);
	SetPack(pack);

	// Add Files to Pack =========================================
	TCHAR seps[] = _T(" \t\n");
	TCHAR* next = nullptr;
	TCHAR* fileName;

	fileName = _tcstok_s(inputFilesName, seps, &next);
	char* data[FILE_CNT];
	DWORD dataSize;

	FILE* file;
	DWORD totalSize = 0;

	while (fileName != NULL)
	{
		RemoveChar(fileName);
		OpenFile(fileName, &file, _T("rb"));
		dataSize = GetFileSize(&file);

		data[pack.pi.fileNum] = static_cast<char*>(malloc(dataSize));
		memset(data[pack.pi.fileNum], _T('\0'), dataSize);

		ReadFile(&file, dataSize, data[pack.pi.fileNum]);
		totalSize += dataSize;
		if (pack.pi.fileNum == 0)
		{
			pack.fi.offset[pack.pi.fileNum] = 0;
		}
		else
		{
			pack.fi.offset[pack.pi.fileNum]
				= pack.fi.offset[pack.pi.fileNum - 1]
				+ pack.fi.fileSize[pack.pi.fileNum - 1];
		}

		memset(pack.fi.fileName[pack.pi.fileNum], _T('\0'), FILE_NAME);
		_tcscpy_s(pack.fi.fileName[pack.pi.fileNum], FILE_NAME, fileName);
		pack.fi.fileSize[pack.pi.fileNum] = dataSize;
		pack.pi.fileNum++;

		CloseFile(&file);

		fileName = _tcstok_s(NULL, seps, &next);
	}

	pack.data = static_cast<char*>(malloc(totalSize));
	memset(pack.data, _T('\0'), totalSize);

	DWORD offset;
	for (DWORD i = 0; i < pack.pi.fileNum; i++)
	{
		offset = pack.fi.offset[i];
		for (DWORD j = 0; j < pack.fi.fileSize[i]; j++)
		{
			pack.data[offset + j] = *(data[i] + j);
		}

		free(data[i]);
	}

	// Write pack to Packfile

	totalSize =
		sizeof(DWORD64) + sizeof(DWORD)			// PACK INFO HEADER
		+ sizeof(DWORD) * FILE_CNT * 2			// FILE INFO HEADER 1
		+ sizeof(TCHAR) * FILE_CNT * FILE_NAME	// FILE INFO HEADER 2
		+ pack.fi.offset[pack.pi.fileNum - 1]	// DATA
		+ pack.fi.fileSize[pack.pi.fileNum - 1];

	char* totalData = static_cast<char*>(malloc(totalSize));
	memset(totalData, '\0', totalSize);

	DWORD idx;

	idx = 0;
	ChangeDataToChar(totalData, idx, pack.pi.type);
	idx += sizeof(DWORD64);
	ChangeDataToChar(totalData, idx, pack.pi.fileNum);
	idx += sizeof(DWORD);
	ChangeDataArrToChar(totalData, pack.fi.fileSize, idx, FILE_CNT);
	idx += sizeof(DWORD) * FILE_CNT;
	ChangeDataArrToChar(totalData, pack.fi.offset, idx, FILE_CNT);
	idx += sizeof(DWORD) * FILE_CNT;

	DWORD tmpIdx = idx;
	char cTmp[FILE_NAME] = { '\0', };
	for (int i = 0; i < FILE_CNT; i++)
	{
		WideCharToMultiByte(CP_ACP, 0, pack.fi.fileName[i],
			FILE_NAME, cTmp, FILE_NAME, NULL, NULL);

		for (int j = 0; j < FILE_NAME; j++)
		{
			totalData[tmpIdx] = cTmp[j];
			tmpIdx++;
		}
	}
	idx += sizeof(TCHAR) * FILE_CNT * FILE_NAME;

	offset;
	for (int i = 0; i < pack.pi.fileNum; i++)
	{
		offset = pack.fi.offset[i];
		for (int j = 0; j < pack.fi.fileSize[i]; j++)
		{
			totalData[idx + offset + j] = pack.data[offset + j];
		}
	}
	free(pack.data);

	FILE* packfile;
	OpenFile(packName, &packfile, _T("wb"));
	WriteFile(&packfile, totalSize, totalData);
	CloseFile(&packfile);

	_tprintf(_T("Pack Complete!\n\n"));
}


void Unpack(TCHAR* packName)
{
	PACKING pack;
	RemoveChar(packName);

	FILE* file;
	OpenFile(packName, &file, _T("rb"));

	DWORD size = GetFileSize(&file);
	char* data = static_cast<char*>(malloc(size));
	memset(data, '\0', size);
	ReadFile(&file, size, data);
	CloseFile(&file);

	DWORD idx = 0;
	DWORD64 type = 0;
	ChangeCharToData(data, idx, type);

	if (type != pack.pi.type)
	{
		_tprintf(_T("It's not pack file!: %ux, %ux\n"), type, pack.pi.type);
		return;
	}

	idx += sizeof(DWORD64);
	ChangeCharToData(data, idx, pack.pi.fileNum);
	idx += sizeof(DWORD);
	ChangeCharToDataArr(data, pack.fi.fileSize, idx, FILE_CNT);
	idx += sizeof(DWORD) * FILE_CNT;
	ChangeCharToDataArr(data, pack.fi.offset, idx, FILE_CNT);
	idx += sizeof(DWORD) * FILE_CNT;

	char seps[] = " \t\n";
	char* next = nullptr;
	char* tok;

	char cTmp[FILE_NAME] = { '\0', };
	for (int i = 0; i < pack.pi.fileNum; i++)
	{
		for (int j = 0; j < FILE_NAME; j++)
		{
			cTmp[j] = *(data + idx + i * FILE_NAME + j);
		}

		MultiByteToWideChar(CP_ACP, MB_PRECOMPOSED, cTmp,
			FILE_NAME, pack.fi.fileName[i], FILE_NAME);
	}

	idx += sizeof(TCHAR) * FILE_CNT * FILE_NAME;

	DWORD dataSize = size - idx;
	pack.data = static_cast<char*>(malloc(dataSize));
	memset(pack.data, '\0', dataSize);

	DWORD offset;
	for (int i = 0; i < pack.pi.fileNum; i++)
	{
		offset = pack.fi.offset[i];
		for (int j = 0; j < pack.fi.fileSize[i]; j++)
		{
			pack.data[offset + j] = data[idx + offset + j];
		}
	}

	free(data);
	
	for (int i = 0; i < pack.pi.fileNum; i++)
	{
		size = pack.fi.fileSize[i];
		char* data = static_cast<char*>(malloc(size));
		memset(data, '\0', size);

		offset = pack.fi.offset[i];
		for (int j = 0; j < size; j++)
		{
			data[j] = *(pack.data + offset + j);
		}

		FILE* file;
		OpenFile(pack.fi.fileName[i], &file, _T("wb"));
		WriteFile(&file, size, data);
		CloseFile(&file);
		free(data);
	}

	free(pack.data);
	_tprintf(_T("Unpack Complete!\n\n"));
}

void AddPack(TCHAR* packName, TCHAR* inputFilesName)
{
	PACKING pack;
	RemoveChar(packName);
	SetPack(pack);

	// ReadPackFile(TCHAR * packname, PACKING & pack)
	
	FILE* packfile;
	OpenFile(packName, &packfile, _T("rb"));

	DWORD size = GetFileSize(&packfile);
	char* data = static_cast<char*>(malloc(size));
	memset(data, '\0', size);
	ReadFile(&packfile, size, data);
	CloseFile(&packfile);

	DWORD idx = 0;
	DWORD64 type = 0;
	ChangeCharToData(data, idx, type);

	if (type != pack.pi.type)
	{
		_tprintf(_T("It's not pack file!\n"));
		return;
	}

	idx += sizeof(DWORD64);
	ChangeCharToData(data, idx, pack.pi.fileNum);
	idx += sizeof(DWORD);
	ChangeCharToDataArr(data, pack.fi.fileSize, idx, FILE_CNT);
	idx += sizeof(DWORD) * FILE_CNT;
	ChangeCharToDataArr(data, pack.fi.offset, idx, FILE_CNT);
	idx += sizeof(DWORD) * FILE_CNT;

	char seps[] = " \t\n";
	char* next = nullptr;
	char* tok;

	char cTmp[FILE_NAME] = { '\0', };
	for (int i = 0; i < pack.pi.fileNum; i++)
	{
		tok = strtok_s(data + idx + i * FILE_NAME, seps, &next);
		strcpy_s(cTmp, FILE_NAME, tok);

		MultiByteToWideChar(CP_ACP, MB_PRECOMPOSED, cTmp,
			FILE_NAME, pack.fi.fileName[i], FILE_NAME);
	}

	idx += sizeof(TCHAR) * FILE_CNT * FILE_NAME;

	DWORD dataSize = size - idx;

	
	//AddFilesToPack
	
	TCHAR tSeps[] = _T(" \t\n");
	TCHAR* tNext = nullptr;
	TCHAR* fileName;

	fileName = _tcstok_s(inputFilesName, tSeps, &tNext);
	char* fileData[FILE_CNT];
	DWORD fileSize;

	FILE* file;
	DWORD originFileNum;

	if (pack.pi.fileNum == 0)
		originFileNum = 0;
	else
		originFileNum = pack.pi.fileNum;

	while (fileName != NULL)
	{
		RemoveChar(fileName);
		OpenFile(fileName, &file, _T("rb"));
		fileSize = GetFileSize(&file);

		fileData[pack.pi.fileNum] = static_cast<char*>(malloc(fileSize));
		memset(fileData[pack.pi.fileNum], _T('\0'), fileSize);

		ReadFile(&file, fileSize, fileData[pack.pi.fileNum]);
		dataSize += fileSize;

		if (pack.pi.fileNum == 0)
		{
			pack.fi.offset[pack.pi.fileNum] = 0;
		}
		else
		{
			pack.fi.offset[pack.pi.fileNum]
				= pack.fi.offset[pack.pi.fileNum - 1]
				+ pack.fi.fileSize[pack.pi.fileNum - 1];
		}

		memset(pack.fi.fileName[pack.pi.fileNum], _T('\0'), FILE_NAME);
		_tcscpy_s(pack.fi.fileName[pack.pi.fileNum], FILE_NAME, fileName);
		pack.fi.fileSize[pack.pi.fileNum] = fileSize;
		pack.pi.fileNum++;

		CloseFile(&file);

		fileName = _tcstok_s(NULL, tSeps, &tNext);
	}

	pack.data = static_cast<char*>(malloc(dataSize));
	memset(pack.data, _T('\0'), dataSize);

	DWORD offset;
	for (int i = 0; i < pack.pi.fileNum; i++)
	{
		offset = pack.fi.offset[i];
		if (i < originFileNum)
		{
			for (int j = 0; j < pack.fi.fileSize[i]; j++)
			{
				pack.data[offset + j] = data[idx + offset + j];
			}
		}	
		else
		{
			for (int j = 0; j < pack.fi.fileSize[i]; j++)
			{
				pack.data[offset + j] = *(fileData[i] + j);
			}
			free(fileData[i]);
		}
	}
	free(data);

	//WritePackToFile

	// 0. Setting
	DWORD totalSize = dataSize +
			( sizeof(DWORD64) + sizeof(DWORD)			// PACK INFO HEADER
			+ sizeof(DWORD) * FILE_CNT * 2				// FILE INFO HEADER 1
			+ sizeof(TCHAR) * FILE_CNT * FILE_NAME);	// FILE INFO HEADER 2


	char* totalData = static_cast<char*>(malloc(totalSize));
	memset(totalData, '\0', totalSize);

	//2. Get pack.header

	idx = 0;
	ChangeDataToChar(totalData, idx, pack.pi.type);
	idx += sizeof(DWORD64);
	ChangeDataToChar(totalData, idx, pack.pi.fileNum);
	idx += sizeof(DWORD);
	ChangeDataArrToChar(totalData, pack.fi.fileSize, idx, FILE_CNT);
	idx += sizeof(DWORD) * FILE_CNT;
	ChangeDataArrToChar(totalData, pack.fi.offset, idx, FILE_CNT);
	idx += sizeof(DWORD) * FILE_CNT;

	DWORD tmpIdx = idx;
	char cTmp2[FILE_NAME] = { '\0', };
	for (int i = 0; i < FILE_CNT; i++)
	{
		WideCharToMultiByte(CP_ACP, 0, pack.fi.fileName[i],
			FILE_NAME, cTmp2, FILE_NAME, NULL, NULL);

		for (int j = 0; j < FILE_NAME; j++)
		{
			totalData[tmpIdx] = cTmp2[j];
			tmpIdx++;
		}
	}
	idx += sizeof(TCHAR) * FILE_CNT * FILE_NAME;

	//2. Get pack.data
	for (int i = 0; i < pack.pi.fileNum; i++)
	{
		offset = pack.fi.offset[i];
		for (int j = 0; j < pack.fi.fileSize[i]; j++)
		{
			totalData[idx + offset + j] = pack.data[offset + j];
		}
	}

	free(pack.data);

	//3. Write Data to File

	OpenFile(packName, &packfile, _T("wb"));
	WriteFile(&packfile, totalSize, totalData);
	CloseFile(&packfile);

	_tprintf(_T("Add Files to Packfile Complete!\n\n"));
}

void ShowFiles(TCHAR* packName)
{
	PACKING pack;
	SetPack(pack);
	RemoveChar(packName);
	
	FILE* file;
	OpenFile(packName, &file, _T("rb"));

	DWORD size = GetFileSize(&file);
	char* data = static_cast<char*>(malloc(size));
	memset(data, '\0', size);
	ReadFile(&file, size, data);
	CloseFile(&file);

	DWORD idx = 0;
	DWORD64 type = 0;
	DWORD fileNum = 0;
	ChangeCharToData(data, idx, type);

	if (type != pack.pi.type)
	{
		_tprintf(_T("It's not pack file!\n"));
		return;
	}
		

	idx += sizeof(DWORD64);
	ChangeCharToData(data, idx, fileNum);
	idx += (sizeof(DWORD) + sizeof(DWORD) * FILE_CNT * 2);

	char seps[] = " \t\n";
	char* next = nullptr;
	char* tok;

	char cTmp[FILE_NAME] = { '\0', };
	TCHAR fileName[FILE_NAME] = { '\0', };
	for (int i = 0; i < fileNum; i++)
	{
		tok = strtok_s(data + idx + i * FILE_NAME, seps, &next);
		strcpy_s(cTmp, FILE_NAME, tok);

		MultiByteToWideChar(CP_ACP, MB_PRECOMPOSED, cTmp,
			FILE_NAME, fileName, FILE_NAME);

		_tprintf(_T("%d. %s\n"), i, fileName);
	}

	_tprintf(_T("\n"));
}


void UnpackFiles(TCHAR* packName, TCHAR* inputFilesName)
{
	PACKING pack;
	SetPack(pack);

	FILE* packfile;
	RemoveChar(packName);
	OpenFile(packName, &packfile, _T("rb"));

	DWORD packSize = GetFileSize(&packfile);
	char* packData = static_cast<char*>(malloc(packSize));
	memset(packData, '\0', packSize);
	ReadFile(&packfile, packSize, packData);
	CloseFile(&packfile);

	DWORD idx = 0;
	DWORD64 type = 0;
	ChangeCharToData(packData, idx, type);

	if (type != pack.pi.type)
	{
		_tprintf(_T("It's not pack file!: %x, %x\n"), type, pack.pi.type);
	}
		
	
	idx += sizeof(DWORD64);
	ChangeCharToData(packData, idx, pack.pi.fileNum);
	idx += sizeof(DWORD);
	ChangeCharToDataArr(packData, pack.fi.fileSize, idx, FILE_CNT);
	idx += sizeof(DWORD) * FILE_CNT;
	ChangeCharToDataArr(packData, pack.fi.offset, idx, FILE_CNT);
	idx += sizeof(DWORD) * FILE_CNT;

	char seps[] = " \t\n";
	char* next = nullptr;
	char* tok;

	char cTmp[FILE_NAME] = { '\0', };
	for (int i = 0; i < pack.pi.fileNum; i++)
	{
		for (int j = 0; j < FILE_NAME; j++)
		{
			cTmp[j] = *(packData + idx + i * FILE_NAME + j);
		}

		MultiByteToWideChar(CP_ACP, MB_PRECOMPOSED, cTmp,
			FILE_NAME, pack.fi.fileName[i], FILE_NAME);
	}

	idx += sizeof(TCHAR) * FILE_CNT * FILE_NAME;

	DWORD dataSize = packSize - idx;

	pack.data = static_cast<char*>(malloc(dataSize));
	memset(pack.data, '\0', dataSize);

	DWORD offset;
	for (int i = 0; i < pack.pi.fileNum; i++)
	{
		offset = pack.fi.offset[i];
		for (int j = 0; j < pack.fi.fileSize[i]; j++)
		{
			pack.data[offset + j] = packData[idx + offset + j];
		}
	}

	free(packData);

	DWORD fileSize;
	bool flag = false;

	TCHAR tSeps[] = _T(" \t\n");
	TCHAR* tNext = nullptr;
	TCHAR* fileName;
	fileName = _tcstok_s(inputFilesName, tSeps, &tNext);

	while (fileName != NULL)
	{
		RemoveChar(fileName);
		for (int i = 0; i < pack.pi.fileNum; i++)
		{
			if (_tcscmp(fileName, pack.fi.fileName[i]) == 0)
			{
				fileSize = pack.fi.fileSize[i];
				char* fileData = static_cast<char*>(malloc(fileSize));
				memset(fileData, '\0', fileSize);

				offset = pack.fi.offset[i];
				for (int j = 0; j < fileSize; j++)
				{
					fileData[j] = *(pack.data + offset + j);
				}

				FILE* file;
				OpenFile(pack.fi.fileName[i], &file, _T("wb"));
				WriteFile(&file, fileSize, fileData);
				CloseFile(&file);
				free(fileData);

				_tprintf(_T("%s is unpacked successfully\n"), fileName);
				flag = true;
				break;
			}
		}

		if (flag == false)
		{
			_tprintf(_T("%s does not exist in the pack file\n"), fileName);
		}

		fileName = _tcstok_s(NULL, tSeps, &tNext);
		flag = false;
	}
	free(pack.data);
	_tprintf(_T("Unpack Selected Files Complete!\n\n"));
}

void UpdateFile(TCHAR* packName, TCHAR* fileName)
{
	// Get file data to update
	FILE* file;
	DWORD fileSize;
	
	RemoveChar(fileName);
	OpenFile(fileName, &file, _T("rb"));
	fileSize = GetFileSize(&file);
	char* fileData = static_cast<char*>(malloc(fileSize));
	ReadFile(&file, fileSize, fileData);
	CloseFile(&file);

	// Get packfile to update
	FILE* packFile;
	DWORD packSize;

	RemoveChar(packName);
	OpenFile(packName, &packFile, _T("rb"));
	packSize = GetFileSize(&packFile);
	char* packData = static_cast<char*>(malloc(packSize));
	ReadFile(&packFile, packSize, packData);
	CloseFile(&packFile);

	// Write packfile to pack
	PACKING pack;
	SetPack(pack);

	DWORD idx = 0;
	DWORD64 type = 0;
	ChangeCharToData(packData, idx, type);

	if (type != pack.pi.type)
	{
		_tprintf(_T("It's not pack file!\n"));
		return;
	}
	
	idx += sizeof(DWORD64);
	ChangeCharToData(packData, idx, pack.pi.fileNum);
	idx += sizeof(DWORD);
	ChangeCharToDataArr(packData, pack.fi.fileSize, idx, FILE_CNT);
	idx += sizeof(DWORD) * FILE_CNT;
	ChangeCharToDataArr(packData, pack.fi.offset, idx, FILE_CNT);
	idx += sizeof(DWORD) * FILE_CNT;


	char cTmp[FILE_NAME] = { '\0', };
	for (int i = 0; i < pack.pi.fileNum; i++)
	{
		for (int j = 0; j < FILE_NAME; j++)
		{
			cTmp[j] = *(packData + idx + i * FILE_NAME + j);
		}

		MultiByteToWideChar(CP_ACP, MB_PRECOMPOSED, cTmp,
			FILE_NAME, pack.fi.fileName[i], FILE_NAME);
	}

	bool flag = false;
	// Update packfile data through pack
	for (int i = 0; i < pack.pi.fileNum; i++)
	{
		if (_tcscmp(fileName, pack.fi.fileName[i]) == 0)
		{
			pack.fi.fileSize[i] = fileSize;
			idx = sizeof(DWORD64) + sizeof(DWORD) 
					+ sizeof(DWORD) * i; 
			ChangeDataToChar(packData, idx, pack.fi.fileSize[i]);

			pack.fi.offset[i]
				= pack.fi.offset[pack.pi.fileNum - 1]
				+ pack.fi.fileSize[pack.pi.fileNum - 1];
			idx = sizeof(DWORD64) + sizeof(DWORD) + sizeof(DWORD) * FILE_CNT
					+ sizeof(DWORD) * i;
			ChangeDataToChar(packData, idx, pack.fi.offset[i]);
			flag = true;
			break;
		}
	}

	if (flag == false)
	{
		_tprintf(_T("%s doesn't exist in the packfile\n"), fileName);
		free(fileData);
		free(packData);
		return;
	}
	// Write pack to packfile
	packSize += fileSize;
	char* newData = static_cast<char*>(malloc(packSize));
	memset(newData, _T('\0'), packSize);
	strcpy_s(newData, packSize, packData);
	strcat_s(newData, packSize, fileData);

	OpenFile(packName, &packFile, _T("wb"));
	WriteFile(&packFile, packSize, newData);
	CloseFile(&packFile);

	free(fileData);
	free(packData);
	free(newData);
	_tprintf(_T("Update Selected File Complete!\n\n"));
}

//=====================================================

void RemoveChar(TCHAR* fileName)
{
	for (; *fileName != '\0'; fileName++)
	{
		if ((_tcscmp(fileName, _T("\\")) == 0) || (_tcscmp(fileName, _T("\"")) == 0) ||
			(_tcscmp(fileName, _T("/")) == 0) || (_tcscmp(fileName, _T(">")) == 0) ||
			(_tcscmp(fileName, _T(":")) == 0) || (_tcscmp(fileName, _T("<")) == 0) ||
			(_tcscmp(fileName, _T("*")) == 0) || (_tcscmp(fileName, _T("?")) == 0))
		{
			_tcscpy_s(fileName, sizeof(fileName + 1), fileName + 1);
			fileName--;
		}
		else if ((_tcscmp(fileName, _T(" ")) == 0) ||
			(_tcscmp(fileName, _T("\t")) == 0))
		{
			_tcscpy_s(fileName, sizeof(_T("_")), _T("_"));
		}
		else if(_tcscmp(fileName, _T("\n")) == 0)
		{
			_tcscpy_s(fileName, sizeof(_T("\0")), _T("\0"));
		}
	}
}

void SetPack(PACKING& pack)
{
	pack.pi.type = 0x99886655;
	pack.pi.fileNum = 0;
}

template <typename T>
void ChangeCharToData(const char* input, const DWORD& idx, T& output)
{
	DWORD tmpIdx = idx;
	DWORD dataSize = sizeof(output);
	DWORD64 flag = 0x000000ff;
	for (DWORD i = 0; i < dataSize; i++)
	{
		output += ((input[tmpIdx] & flag) << (i * 8));
		tmpIdx++;
	}
}

template <typename T>
void ChangeCharToDataArr(const char* input, T& output, const DWORD& idx, const DWORD& cnt)
{
	DWORD tmpIdx = idx;
	DWORD dataSize = sizeof(output[0]);
	DWORD64 flag = 0x000000ff;
	for (DWORD i = 0; i < cnt; i++)
	{
		for (DWORD j = 0; j < dataSize; j++)
		{
			output[i] += ((input[tmpIdx] & flag) << (j * 8));
			tmpIdx++;
		}
	}
}

template <typename T>
void ChangeDataToChar(char* output, const DWORD& idx, const T& input)
{
	DWORD tmpIdx = idx;
	DWORD dataSize = sizeof(input);
	DWORD64 flag = 0x000000ff;

	for (DWORD i = 0; i < dataSize; i++)
	{
		output[tmpIdx] = ((input >> (i * 8)) & flag);
		tmpIdx++;
	}
}

template <typename T>
void ChangeDataArrToChar(char* output, const T& input, const DWORD& idx, const DWORD& cnt)
{
	DWORD tmpIdx = idx;
	DWORD dataSize = sizeof(input[0]);
	DWORD64 flag = 0x000000ff;

	for (DWORD i = 0; i < cnt; i++)
	{
		for (DWORD j = 0; j < dataSize; j++)
		{
			output[tmpIdx] = ((input[i] >> (j * 8)) & flag);
			tmpIdx++;
		}
	}

}
```