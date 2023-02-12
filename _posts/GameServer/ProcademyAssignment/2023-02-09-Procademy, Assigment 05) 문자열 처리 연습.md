---
title: Procademy, Assignment 05) 문자열 처리 연습
categories: ProcademyAssignment
tags: 
toc: true
toc_sticky: true
---

## **과제**

문자열을 비교하여 찾아주는 간단한 사전을 구현하는 과제. 이 과제의 목표는 문자열 처리 함수들에 익숙해지는 것이다. 

<br/>

## **과제 코드**

```c++
#include <iostream>
#include <windows.h>
#include <tchar.h>

#define DICT_SIZE 10
#define LANG_CNT 2
#define WORD_MAX 10

#define IN_MAX 200
#define OUT_MAX 400

char dict[DICT_SIZE][LANG_CNT][WORD_MAX] =
{
	{"i", "나"},
	{"you", "너"},
	{"am", "은(는)"},
	{"is", "은(는)"},
	{"are", "은(는)"},
	{"boy", "소년"},
	{"girl", "소녀"},
	{"a", "하나의"},
	{"man", "남자"},
	{"woman", "여자"}
};

void Translate()
{
	bool found = false;
	int buffer_idx = 0;

	while(1)
	{
		char input[IN_MAX] = "";
		char buffer[IN_MAX] = "";
		char output[OUT_MAX] = "";

		// input section
		printf("100자 이내 영어 문장을 입력해주세요.\n");
		scanf_s("%[^\n]s", &input, IN_MAX);
		_strlwr_s(input);
		rewind(stdin);

		for (int i = 0; i <= (int)strlen(input); i++)
		{
			// logic section
			// 1. tokenize
			if ( input[i] != ' ' && input[i] != '\0')
			{
				buffer[buffer_idx] = input[i];
				buffer_idx++;
			}
			else
			{
				buffer[buffer_idx] = '\0';

				//// 2. translate
				for (int dict_idx = 0; dict_idx < DICT_SIZE; dict_idx++)
				{
					if (strcmp(dict[dict_idx][0], buffer) == 0)
					{
						strcat_s(output, dict[dict_idx][1]);
						strcat_s(output, " ");
						found = true;
						break;
					}
				}

				if (found == false)
				{
					strcat_s(output, buffer);
					strcat_s(output, " ");
				}

				buffer_idx = 0;
				found = false;
			}
		}
		
		// render section
		printf("%s\n\n", output);
		buffer_idx = 0;
	}
}

int _tmain(int argc, _TCHAR* argv[])
{
	Translate();
	return 0;
}

```
