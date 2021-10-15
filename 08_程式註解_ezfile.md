---
title : 08_ezfile程式說明
---
# ezfile程式說明
by : CY,Tsai
Date : 2021/10/15

---
# 壹、簡介
這個程式實作幾個常用到的檔案讀寫刪除函數。因為完整的讀取需要使用CreateFile()、SetFilePointerEx()、ReadFile()、CloseHandle()，寫入也是一樣，中間有一大堆參數傳遞，所以事先寫好這些常用的功能可以節省很多時間。

# 貳、標頭檔
```cpp=
#pragma once

#include <Windows.h>
#include "../config.h"
#include "hexdump.h"

BOOL ReadBuffer(
	LPCTSTR,
	LARGE_INTEGER,
	DWORD,
	PUCHAR,
	ULONG,
	PULONG);

BOOL WriteBuffer(
	LPCTSTR,
	LARGE_INTEGER,
	DWORD,
	PUCHAR,
	ULONG,
	PULONG);

BOOL UpdateFileAttributes(
	LPCTSTR,
	DWORD,
	BOOL
);

BOOL DeleteFileZero(
	LPCTSTR
);

BOOL FakeDeleteFile(
	LPCTSTR
);
```
標頭檔定義五個函數，分別為讀取、寫入、更新檔案屬性、假刪除(徹底)。

## 參、實作
### 01. 讀取檔案進記憶體
```cpp=9
BOOL ReadBuffer(
	LPCTSTR lpFileName,
	LARGE_INTEGER liDistanceToMove,
	DWORD dwMoveMethod,
	PUCHAR pbBuffer,
	ULONG cbBuffer,
	PULONG pcbResult)
{
	HANDLE hFile;
	LARGE_INTEGER liNewFilePointer;
	BOOL bResult = FALSE;

	if ((hFile = CreateFile(
		lpFileName,
		GENERIC_READ,
		FILE_SHARE_READ,
		NULL,
		OPEN_EXISTING,
		FILE_ATTRIBUTE_NORMAL,
		NULL
	)) == INVALID_HANDLE_VALUE) {
		DEBUG("open %s fails: %d\n",
			lpFileName, GetLastError());
		return FALSE;
	}

	if (!SetFilePointerEx(
		hFile,
		liDistanceToMove,
		&liNewFilePointer,
		dwMoveMethod)) {
		DEBUG("set pointer %s fails: %d\n",
			lpFileName, GetLastError());
		goto Error_Exit;
	}

	if (!(bResult = ReadFile(
		hFile,
		pbBuffer,
		cbBuffer,
		pcbResult,
		0))) {
		DEBUG("Read %s error: %d\n",
			lpFileName, GetLastError());
	}
	bResult = TRUE;
Error_Exit:
	CloseHandle(hFile);
	return bResult;
}
```
ReadBuffer函數要結合SetFilePointerEx()和ReadFile()，使用SetFilePointerEx的目的是要達到可以從檔案的任意位置開始讀取，而ReadFile顧名思義是用來讀取檔案的。以下是函數定義:

> lpFileName : 檔案名稱，不同於一般函數放HANDLE，節省了事先需要CreateFile()的動作
> liDistanceToMove : 這是可以移動的byte數量，範圍是LONG
> dwMoveMethod : 有三個選項，分別是FILE_BEGIN、FILE_CURRENT、FILE_END
> pbBuffer : 讀取後的存放位置
> cdBuffer : 讀取的最大數量
> pcbBuffer : 實際讀取大小, 會有這個參數的原因是不可能每次讀取都剛剛好到達最大數量，
>             也沒有必要把讀取的資料放大，所以還是要有這個參數確定實際大小。
> return : 只要非0就是成功。錯誤可以使用GetLastError()取得。

---
### 02. 將記憶體寫入檔案
```cpp=60
BOOL WriteBuffer(
	LPCTSTR lpFileName,
	LARGE_INTEGER liDistanceToMove,
	DWORD dwMoveMethod,
	PUCHAR pbBuffer,
	ULONG cbBuffer,
	PULONG pcbResult)
{
	HANDLE hFile;
	LARGE_INTEGER liNewFilePointer;
	BOOL bResult = FALSE;

	if ((hFile = CreateFile(
		lpFileName,
		GENERIC_WRITE,
		FILE_SHARE_READ,
		NULL,
		CREATE_ALWAYS,
		FILE_ATTRIBUTE_NORMAL,
		NULL
	)) == INVALID_HANDLE_VALUE) {
		DEBUG("Open file %s error\n",
			lpFileName);
		return FALSE;
	}

	if (!SetFilePointerEx(
		hFile,
		liDistanceToMove,
		&liNewFilePointer,
		dwMoveMethod)) {
		DEBUG("set pointer %s fails: %d\n",
			lpFileName, GetLastError());
		goto Error_Exit;
	}

	if (!(bResult = WriteFile(
		hFile,
		pbBuffer,
		cbBuffer,
		pcbResult,
		0
	))) {
		DEBUG("Write %s error\n",
			lpFileName);
	}
	bResult = TRUE;
Error_Exit:
	CloseHandle(hFile);
	return bResult;
}
```
將記憶體內容寫入檔案也是一樣的步驟，開啟檔案、移動檔案指標、寫入檔案、關閉檔案。雖然就這四個步驟，但是需要超過40行來描述，所以寫成一個函數是有必要的。以下是函數定義:
> lpFileName : 檔案名稱，而不是HANDLE
> liDistanceToMove : 要移動的byte數
> dwMoveMethod : 同上述的三個選項
> pbBuffer : 要寫入的資料存放位址
> cbBuffer : 寫入的最大數量
> pcbBuffer : 實際寫入大小
> return : 只要非0就是成功。錯誤可以使用GetLastError()取得。


### 03. 小插曲
要如何知道寫入什麼?讀取什麼?以下兩個函數是針對讀取和寫入的dbug函數。
```cpp=
BOOL ReadFile_DEBUG(
    HANDLE       hFile, 
    LPVOID       lpBuffer, 
    DWORD        nNumberOfBytesToRead, 
    LPDWORD      lpNumberOfBytesRead, 
    LPOVERLAPPED lpOverlapped) {
    BOOL retval = ReadFile(hFile, lpBuffer, nNumberOfBytesToRead,
                            lpNumberOfBytesRead, lpOverlapped);
    DEBUG("Read %d\n", nNumberOfBytesToRead);
    hexdump((PUCHAR)lpBuffer, nNumberOfBytesToRead);
    return retval;
}
BOOL WriteFile_DEBUG(
    HANDLE       hFile, 
    LPVOID       lpBuffer, 
    DWORD        nNumberOfBytesToWrite, 
    LPDWORD      lpNumberOfBytesWritten, 
    LPOVERLAPPED lpOverlapped) {
    DEBUG("Write %d\n", nNumberOfBytesToWrite);
    hexdump((PUCHAR)lpBuffer, nNumberOfBytesToWrite);
    return ReadFile(hFile, lpBuffer, nNumberOfBytesToWrite,
                    lpNumberOfBytesWritten, lpOverlapped);
}
```

### 04. 將檔案徹底刪除
有時候因為某些原因需要把檔案徹底刪除，例如勒索程式加密後的金鑰，早期勒索程式有些就是沒有徹底刪除金鑰被工程師在記憶體中找到進行解密。其中一種刪除的方式為用覆寫模式開啟檔案，然後植入亂碼，最後一步再進行刪除，這樣就算被軟體救回來也沒辦法使用。
```cpp=112
BOOL DeleteFileZero(LPCTSTR lpFileName)
{
	HANDLE hFile;
	BOOL bResult = TRUE;
	PUCHAR abBuffer[4096];
	LARGE_INTEGER FileSize;
	if ((hFile = CreateFile(
		lpFileName,
		GENERIC_WRITE,
		FILE_SHARE_READ,
		NULL,
		CREATE_ALWAYS,
		FILE_ATTRIBUTE_NORMAL,
		NULL
	)) == INVALID_HANDLE_VALUE) {
		DEBUG("Open file %s error\n",
			lpFileName);
		goto Error_Exit;
	}
	GetFileSizeEx(hFile, &FileSize);
	ZeroMemory(abBuffer, sizeof(abBuffer));
	for (LONGLONG p = 0;
		p < FileSize.QuadPart;
		p += sizeof(abBuffer)) {
		DWORD cbBuffer =
			(DWORD)(p < sizeof(abBuffer) ?
				p : sizeof(abBuffer));
		if (!(bResult = WriteFile(
			hFile,
			abBuffer,
			cbBuffer,
			NULL,
			0
		))) {
			DEBUG("Write %s error\n",
				lpFileName);
			break;
		}
	}
	CloseHandle(hFile);
Error_Exit:
	DeleteFile(lpFileName);
	return TRUE;
}
```
這邊就是單純把檔案開啟後，全部覆蓋成0，然後關閉刪除。參數只有檔案名稱。

### 05. 更新檔案屬性
如果要增加或減少檔案的屬性，需要把原來檔案的屬性讀出來，然後用OR或AND結合，再傳回檔案。而這個函數可以像SetFileAttributes()一樣，一個指令完成。
```cpp=157
BOOL UpdateFileAttributes(
	LPCTSTR lpFileName,
	DWORD dwFileAttributes,
	BOOL bFlag
)
{
	BOOL bResult = TRUE;
	DWORD dwAttrs = GetFileAttributes(lpFileName);
	if (dwAttrs == INVALID_FILE_ATTRIBUTES) {
		return FALSE;
	}
	if (bFlag) {
		if (!(dwAttrs & dwFileAttributes))
		{
			bResult = SetFileAttributes(lpFileName,
				dwAttrs | dwFileAttributes);
		}
	}
	else {
		if (dwAttrs & dwFileAttributes)
		{
			DWORD dwNewAttrs = dwAttrs &
				~dwFileAttributes;
			bResult = SetFileAttributes(lpFileName,
				dwNewAttrs);
		}
	}
	return bResult;
}
```
函數只是結合兩個原來定義好的函數，相關說明在另一篇有提到，這裡就不多贅述了。

### 06. 假刪除
假刪除就是沒有真正從電腦裡消失，而是透過一些手法隱藏起來了。這裡用的方式是將檔名修改後使用UpdateFileAttributes加入隱藏屬性。
```cpp=208
BOOL FakeUndeleteFile(LPCTSTR lpFileName)
{
	TCHAR newFileName[MAX_PATH];
	TCHAR* suffix;
	BOOL bResult = FALSE;
	_tcscpy_s(newFileName, lpFileName);
	if (!(suffix = _tcsrchr(newFileName, _T('.')))) {
		return TRUE;
	}
	if (!_tcsnicmp(suffix, FAKEDELETESUFFIX, 8)) {
		*suffix = 0;
		bResult = MoveFile(lpFileName, newFileName);
	}
	if (bResult) {
		bResult = UpdateFileAttributes(lpFileName,
			FILE_ATTRIBUTE_HIDDEN, FALSE);
	}
	return bResult;
}
```
裡面用到的函數都是之前有說明過的，可以看看前幾篇說明文件。