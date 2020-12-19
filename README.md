# Cho-Hyeon-Min

#pragma warning(disable:4996)
#include <conio.h>             
#include <time.h>
#include <string.h>
#include <process.h>
#include <stdio.h> //printf를 사용하기 위한 헤더파일
#include <stdlib.h> //Sleep()사용을 위한 헤더파일
#include <winsock2.h> //socket을 사용하기 위한 헤더파일

#define BUF_SIZE 1000
void ErrorHandling(char *message);

int main(int argc, char *argv[])
{
	WSADATA wsaData;
	SOCKET servSock;

	char data[BUF_SIZE]; // BUF_SIZE 만큼 data 설정
	int seq[BUF_SIZE]; // BUF_SIZE 만큼 seq 설정

	int strLen;  //data   recv 크기 저장 변수
	int clntAdrSz;
	int i; // 반복문제어 변수
	int mode; // 혼잡제어 설정변수 

	int windowSize = 1, start = 0; // windowSize 처음 시작은 1, start 0
	SOCKADDR_IN   servAdr, clntAdr;
	if (argc != 2) {
		printf("Usage : %s <port>\n", argv[0]);
		exit(1);
	}
	if (WSAStartup(MAKEWORD(2, 2), &wsaData) != 0)
		ErrorHandling("WSAStartup() error!");

	servSock = socket(PF_INET, SOCK_DGRAM, 0);
	if (servSock == INVALID_SOCKET)
		ErrorHandling("UDP socket creation error");

	memset(&servAdr, 0, sizeof(servAdr));
	servAdr.sin_family = AF_INET;
	servAdr.sin_addr.s_addr = htonl(INADDR_ANY);
	servAdr.sin_port = htons(atoi(argv[1]));

	if (bind(servSock, (SOCKADDR*)&servAdr, sizeof(servAdr)) == SOCKET_ERROR)
		ErrorHandling("bind() error");

	while (1)
	{
		clntAdrSz = sizeof(clntAdr);
		recvfrom(servSock, &windowSize, 4, 0, (SOCKADDR*)&clntAdr, &clntAdrSz);	                   //windowSize recv
		recvfrom(servSock, &mode, 4, 0, (SOCKADDR*)&clntAdr, &clntAdrSz);
		//mode recv

		for (i = 0; i < windowSize; i++) {
			strLen = recvfrom(servSock, data, 1, 0, (SOCKADDR*)&clntAdr, &clntAdrSz);		//data windowSize만큼 recv
			data[strLen] = 0;
			printf("%s", data);										//recv한 데이터 출력
		}
		printf("\n");

		strLen = recvfrom(servSock, seq, windowSize * 4, 0, (SOCKADDR*)&clntAdr, &clntAdrSz);
		//seq windowSize*4(sizeof(int))만큼 recv

		if (mode == 3 && windowSize > 30) {
			//중복ACK 시나리오 선택, windowSize>30이상 일 때 실행
			for (i = 0; i < windowSize; i++) {
				//수신한 모든 seq 마지막으로   전송 성공한 seq로 변경
				seq[i] = seq[0];
			}
			mode = 1;		//정상 시나리오로 변경
		}
		printf("<----------FromClient: %s  \n", data);		//recv한 데이터 출력
		printf("ToClient------------> : ");
		for (i = 0; i < windowSize; i++) {
			printf(" %d", seq[i]);
		}
		printf("\n\n");
		Sleep(1500);
		if (mode == 2 && windowSize > 30) {
			Sleep(3000);
		}
		else {
			sendto(servSock, seq, windowSize * 4, 0, (SOCKADDR*)&clntAdr, clntAdrSz);
			//recv한   seq send
		}
		if (start > 500) {
			break;
		}
	}

	closesocket(servSock);
	WSACleanup();
	return 0;
}

void ErrorHandling(char *message)
{
	fputs(message, stderr);
	fputc('\n', stderr);
	exit(1);
}
