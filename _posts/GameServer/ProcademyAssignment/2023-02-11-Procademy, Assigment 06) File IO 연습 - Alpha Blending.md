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
#include <windows.h>

void AlphaBlending()
{
	int size;
	char fileName1[] = "sample.bmp";
	char fileName2[] = "sample2.bmp";
	char resultFileName[] = "result.bmp";

	errno_t ret;
	FILE* buffer1 = nullptr;
	FILE* buffer2 = nullptr;
	FILE* result = nullptr;
	BITMAPFILEHEADER bf;
	BITMAPINFOHEADER bi;

	// 1. Read Bitmap File

	// 1) Get File Header
	ret = fopen_s(&buffer1, fileName1, "rb");
	if (ret != 0)
		printf("Fail to open %s : %d\n", fileName1, ret);

	fread(&bf, sizeof(bf), 1, buffer1);
	fread(&bi, sizeof(bi), 1, buffer1);
	size = (bi.biWidth * bi.biHeight * bi.biBitCount) / 8;

	char* pImageBuffer1 = static_cast<char*>(malloc(size));
	char* pImageBuffer2 = static_cast<char*>(malloc(size));
	char* pImageBlend = static_cast<char*>(malloc(size));

	// 2) Get File1 Data
	fread(pImageBuffer1, size, 1, buffer1);
	fclose(buffer1);

	// 3) Get File2 Data
	ret = fopen_s(&buffer2, fileName2, "rb");
	if (ret != 0)
		printf("Fail to open %s : %d\n", fileName2, ret);

	fread(pImageBuffer2, size, 1, buffer2);
	fclose(buffer2);
	
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
	ret = fopen_s(&result, resultFileName, "wb");
	if (ret != 0)
		printf("Fail to open %s : %d\n", resultFileName, ret);

	fwrite(&bf, sizeof(bf), 1, result);
	fwrite(&bi, sizeof(bi), 1, result);
	fwrite(pImageBlend, size, 1, result);
	fclose(result);

	free(pImageBlend);
	free(pImageBuffer1);
	free(pImageBuffer2);
}


int main()
{
	AlphaBlending();
	return 0;
}
```
