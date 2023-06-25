---
title: Procademy, Q1, 13) FileIO 
categories: ProcademyReview
tags: 
toc: true
toc_sticky: true
---

이 포스트는 프로카데미 (게임 서버 아카데미) 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. file IO**

C 기반으로 파일 입출력을 할 때는 읽을 수 있는 만큼 한번에 많이 읽는 게 성능 상 낫다. 파일 크기 만큼 동적 할당을 받아서 메모리에 다 올린 다음 메모리에서 처리하는 것이 여러번 나누어 읽고 쓰는 것보다 더 효율적이다. 

# **2. fseek** 

파일 입출력 작업을 하면 내부적으로 입출력을 위한 position이 관리 되며 fseek을 통해 이를 조정할 수 있다. 

```
fseek(파일 포인터, 어느정도 단위로 이동할 것인지, 어떻게 이동할 것인지 설정)
```

만약 fseek(pFile, 10, SEEK_CUR)로 한다면 파일의 10byte 지점부터 읽기 시작하겠다는 의미가 된다. 이를 통해 파일 포지션을 지정할 수 있다. 반면 ftell은 파일 포지션을 반환해준다. 이들을 이용해 아래와 같이 파일 사이즈를 얻어낼 수 있다. 그 예제는 아래와 같다. 

```c++
fseek(pFile, 0, SEEK_END);
int size = ftell(pFile);
char* pBuffer = (char*)malloc(size);
fread(pBuffer, size, 1, pFile);
```

이렇게 되면 포지션을 맨 뒤로 설정했기 때문에 크기는 쉽게 구할 수 있지만 읽어올 수는 없다. 따라서 아래처럼 파일 포지션 위치를 바꿔주어야 한다. 

```c++
fseek(pFile, 0, SEEK_END);
int size = ftell(pFile);
fseek(pFile, 0, SEEK_SET); // 혹은 rewind를 사용할 수도 있다.
char* pBuffer = (char*)malloc(size);
fread(pBuffer, size, 1, pFile);
```

# **3. 파일의 구조 - bitmap** 

서버에서 이미지 파일을 직접적으로 사용하는 일은 많이 없다. 하지만 툴을 만들 때 이미지를 데이터로 활용할 수 있다면 작업하기가 수월해진다. bmp은 bitmap 파일로, 파일 안에 bitmap에 대한 헤더가 포함되어 있다. 실제 bmp 파일의 구조는 아래와 같다. 

_______________________
| BITMAP FILE HEADER  |
| BITMAP INFO HEADER  |
|_____________________|
|			          |
|	   IMAGE	      |
|			          |
|			          |
|_____________________|

윈도우에서 bmp 헤더를 정의한 구조는 아래와 같다. 

```c++
typedef struct tagBITMAPFILEHEADER
{
WORD bfType; // BM ( = bitmap)
DWORD bfSize; // 파일의 크기 (이미지의 크기 X)

 // 규격 호환을 위해 예약한 영역 (사용 x)
WORD bfReserved1;
WORD bfReserved2;

// image 파일 시작 위치 지정, (DWORD = 4byte)
DWORD bfOffBits;

} BITMAPFILEHEADER, *LPBITMAPFILEHEADER, *PBITMAPFILEHEADER;
```
BITMAPFILEHEADER는 파일에 대한 내용을 담고 있다. 모든 파일 규격의 맨 앞단에는 파일 Type을 구분할 수 있는 코드를 넣으며 bmp의 경우 BM이 들어간다. 어쩔 수 없이 헤더값에 의존해야 되는 정보에만 의존하고 bfSize, bfOffBits와 같이 우리가 알 수 있는 정보는 직접 읽어오는 것을 권장한다. 

```c++
typedef struct tagBITMAPINFOHEADER
{
DWORD biSize; 
// BITMAPINFOHEADER 구조체의 크기. 호환성을 위해 존재.

	LONG biWidth;
LONG biHeight;
WORD biPlanes; 
// 높이 너비 레이어
// bitmap은 레이어가 없어서 biPlanes에 1이 들어간다.

WORD biBitCount; 
// 한 픽셀의 용량.
// 8-bit는 256가지 색을, 16-bit는 65000가지 색을 표현
// 24, 32 bit는 rgb 각각 1byte 0-255로 동일. 대신 32bit는 마스킹이 용이
// 최근에는 거의 32bit. 용량은 크지만 계산 속도가 조금 더 빠르다.

DWORD biCompression;
// 현재 대부분의 bmp는 압축을 안함

DWORD biSizeImage; // image 크기
LONG biXPelsPerMeter;
LONG biYPelsPerMeter;
// Meter 당 pixel. 인쇄시 사용

DWORD biClrUsed;
DWORD biClrImportant;
// 8-bit에서 빠레뜨 정보를 담기 위해 사용
// 16-bit 이상에서는 사용하지 않는다.

} BITMAPINFOHEADER, *LPBITMAPINFOHEADER, *PBITMAPINFOHEADER;
```

biSize를 통해 BITMAPINFOHEADER 구조체의 크기를 불러올 수 있다. 이를 통해 bmp가 확장된 상황에서도 기존의 프로그램들과 호환이 가능하게끔 대비할 수 있다. 위에서 reserved를 사용한 것과 동일한 이유. 그렇다면 모르는 버전을 만났을 때 구조체가 어떤 구조로 되어있는지 몰라도 이미지가 어디서 시작하는지는 알 수 있다. 상단은 기존 버전을 따라가고 업데이트 된 것은 하단에 붙임으로써 아는 곳까지만 해석하고 건너뛴다. 이러면 새로운 기능은 사용할 수 없어도 기존 기능대로는 사용할 수 있게 된다. 네트워크 헤더 역시 이와 같은 방식을 많이 사용한다. 

# **4. Alpha Blending** 

bitmap image의 픽셀값의 경우 좌표 체계의 문제로 인해 그림의 상하가 뒤집어져있으니 유의해야 한다. rgb도 확인해보면 bgr 순서로 위치한다. 두 파일을 알파블렌딩 해서 새로운 파일을 만드는 예제는 아래와 같다. 

```c++
// 이미지 열기
BITMAPFILEHEADER BmpFileHeader;
BITMAPINFOHEADER BmpInfoHeader;
error_t ret;
int size;

fopen_s(~~~)
ret = fread(~~~);
if(ret!=1) { printf(“헤더 읽기 오류\n”);

// 이미지의 실제 크기에 맞게 할당
size = ( BmpInfoHeader.biWidth * BmpInfoHeader.biHeight 
* BmpInfoHeader.biBitCount ) / 8; 
char* pImageBuffer1 = malloc(size);  
char* pImageBuffer2 = malloc(size);
ret = fread(pImageBuffer1, size, 1, pImageFile);
if(ret!=1) { printf(“이미지 읽기 오류\n”); // Buffer2도 동일하게…

// 이미지 합치기 (픽셀 단위로 더하기)
char* pImageBlend = malloc(size);
DWORD *pDest = (DWORD*)pImageBlend;
DWORD* pSrc1 = (DWORD*)pImageBuffer1;
DWORD* pSrc2 = (DWORD*)pImageBuffer2;

for( … h < biHeight ; h++)
	for( … w < biWidth ; w++)

		*pDest = (*pSrc1/2 + *pSrc2/2) ;
pDest++;
pSrc++;

// 새 bmp 파일에 기존 데이터 덮어쓰기
fwrite(BmpFileHeader, sizeof(BmpFileHeader), 1, pBlendFile);
fwrite(BmpInfoHeader, sizeof(BmpInfoHeader), 1, pBlendFile);
fwrite(pImageBlend, size, 1, pBlendFile);

// 해제
free(pImageBlend);
free(pImageBuffer1);
free(pImageBuffer2);
fclose();
```
그러나 위와 같이 코드를 실행하면 이상한 색이 된다.
```c++
for( … h < biHeight ; h++)
	for( … w < biWidth ; w++)

		*pDest = (*pSrc1/2 + *pSrc2/2) ;
pDest++;
pSrc++;
```
/2 는 right shift나 다름이 없는데, 그럼 R의 1bit가 G의 최상위 비트로, G의 최상위 비트가 B의 최상위 비트로 가게 된다. 픽셀이 RGB에 대한 데이터를 모두 한번에 담고 있기 때문이다. 따라서 픽셀 단위가 아니라 픽셀의 바이트 단위로 나누어서 연산을 해야 의도한 결과가 나오게 된다.

```c++
BYTE a1 = *((BYTE*) pSrc1 + 3);
BYTE r1 = *((BYTE*) pSrc1 + 2);
BYTE g1 = *((BYTE*) pSrc1 + 1);
BYTE b1 = *((BYTE*) pSrc1 + 0);
…

*((BYTE*)pDest + 0) = b1/2 + b2/2;
..

pDest++;
pSrc++;
```
이를 다소 무식한 방법으로 해결한다면 위와 같이 해결할 수 있다. BYTE 단위로 하나씩 뜯어내서 연산해주는 것이다. 

```c++
BYTE a1 = (*pSrc1 & 0x000000ff) >> 24;
BYTE r1 = (*pSrc1 & 0x000000ff) >> 16;
BYTE g1 = (*pSrc1 & 0x000000ff) >> 8;
BYTE b1 = *pSrc1 & 0x000000ff;
…

pDest++;
pSrc1++;
pSrc2++;
```
마스킹을 사용하면 더 가독성 있게 해결할 수도 있다. 합치는 경우도 bit or 을 사용하면 더 보기 좋다. 그러나 사실 가장 단순한 방법은 아래처럼 rgb의 최상위 비트를 0으로 마스킹해주는 것이다. 

```c++
*pDest = ((*pSrc1 / 2) & 0x7f7f7f7f) + ((*pSrc2 / 2) & 0x7f7f7f7f)
```
이 방법의 경우 rgb 비율을 우리가 원하는대로 조절할 수 없고, 5:5만 가능하다는 한계가 있다.



