---
title: Procademy, Assignment 06) File IO 연습 - Alpha Blending
categories: ProcademyAssignment
tags: 
toc: true
toc_sticky: true
---

## **과제**

bmp 파일 두개를 불러와서 알파 블렌딩하고 그 결과물 파일을 저장한다. 이를 통해 파일 입출력에 익숙해지는 것이 목표이다. 

<br/>

## **과제 코드**

```c++
#include <iostream>
#include <stdlib.h>
#include <windows.h>

void OpenFile_rb(char filename[], FILE** file);
void OpenFile_wb(char filename[], FILE** file);
void CloseFile(FILE** file);

void ReadBmpHeader(FILE** file, BITMAPFILEHEADER* bf, BITMAPINFOHEADER* bi);
void ReadBmpData(FILE** file, int size, char* output);
void WriteBmpFile(FILE** output, BITMAPFILEHEADER* bf, BITMAPINFOHEADER* bi, int size, char* data);

void AlphaBlending();

int main()
{
	AlphaBlending();
	return 0;
}

void AlphaBlending()
{
	int size;
	char fileName1[] = "sample.bmp";
	char fileName2[] = "sample2.bmp";
	char resultFileName[] = "result.bmp";

	FILE* buffer1 = nullptr;
	FILE* buffer2 = nullptr;
	FILE* result = nullptr;
	BITMAPFILEHEADER bf;
	BITMAPINFOHEADER bi;

	// 1. Read Bitmap File

	// 1) Get File Header
	OpenFile_rb(fileName1, &buffer1);
	ReadBmpHeader(&buffer1, &bf, &bi);
	size = (bi.biWidth * bi.biHeight * bi.biBitCount) / 8;

	char* pImageBuffer1 = static_cast<char*>(malloc(size));;
	char* pImageBuffer2 = static_cast<char*>(malloc(size));;
	char* pImageBlend = static_cast<char*>(malloc(size));;

	// 2) Get File1 Data
	ReadBmpData(&buffer1, size, pImageBuffer1);
	CloseFile(&buffer1);

	// 3) Get File2 Data
	OpenFile_rb(fileName2, &buffer2);
	ReadBmpData(&buffer2, size, pImageBuffer2);
	CloseFile(&buffer2);
	
	// 2. Write File data

	// 1) Get Blending Data
	DWORD* pDest = (DWORD*)pImageBlend;
	DWORD* pSrc1 = (DWORD*)pImageBuffer1;
	DWORD* pSrc2 = (DWORD*)pImageBuffer2;

	for (int h = 0; h < bi.biHeight; h++)
	{
		for (int w = 0; w < bi.biWidth; w++)
		{
			*pDest = ((*pSrc1 / 2) & 0x7f7f7f7f) + ((*pSrc2 / 2) & 0x7f7f7f7f);
			pDest++;
			pSrc1++;
			pSrc2++;
		}
	}

	// 2) Write Bmp File
	OpenFile_wb(resultFileName, &result);
	WriteBmpFile(&result, &bf, &bi, size, pImageBlend);
	CloseFile(&result);

	free(pImageBlend);
	free(pImageBuffer1);
	free(pImageBuffer2);

}

void OpenFile_rb(char filename[], FILE** file)
{
	errno_t ret = fopen_s(&(*file), filename, "rb");
	if (ret != 0)
		printf("Fail to open %s : %d\n", filename, ret);
}

void OpenFile_wb(char filename[], FILE** file)
{
	errno_t ret = fopen_s(&(*file), filename, "wb");
	if (ret != 0)
		printf("Fail to open %s : %d\n", filename, ret);
}

void CloseFile(FILE** file)
{
	errno_t ret = fclose(*file);
	if (ret != 0)
		printf("Fail to close file : %d\n", ret);
}


void ReadBmpHeader(FILE** file, BITMAPFILEHEADER* bf, BITMAPINFOHEADER* bi)
{
	errno_t ret;

	fread(bf, sizeof(*bf), 1, *file);
	ret = ferror(*file);

	if (ret != 0)
		printf("Fail to read file header : %d\n", ret);

	fread(bi, sizeof(*bi), 1, *file);
	ret = ferror(*file);

	if (ret != 0)
		printf("Fail to read info header : %d\n", ret);
}

void ReadBmpData(FILE** file, int size, char* data)
{
	errno_t ret;
	fread(data, size, 1, *file);
	ret = ferror(*file);

	if (ret != 0)
		printf("Fail to read data : %d\n", ret);
}

void WriteBmpFile(FILE** output, BITMAPFILEHEADER* bf,
	BITMAPINFOHEADER* bi, int size, char* data)
{
	errno_t ret;

	fwrite(bf, sizeof(*bf), 1, *output);
	ret = ferror(*output);

	if (ret != 0)
		printf("Fail to write file header : %d\n", ret);

	fwrite(bi, sizeof(*bi), 1, *output);
	ret = ferror(*output);

	if (ret != 0)
		printf("Fail to write info header : %d\n", ret);

	fwrite(data, size, 1, *output);
	ret = ferror(*output);

	if (ret != 0)
		printf("Fail to write data : %d\n", ret);
}
```
