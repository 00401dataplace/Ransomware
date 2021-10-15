---
title : 07_DECQUEUE
---
# DECQUEUE程式說明
by : CY, Tsai
Date : 2021/10/15

---

## 壹、簡介
這個程式主要用來解密檔案，但是不同於一般的解密，因為要與視窗進行訊息傳遞，解密後的檔名要傳到視窗顯示，如果用解一個檔案就傳一次檔名給視窗的方式，會浪費很多時間。從訊息傳遞就可以知道，視窗與解密程式是兩個不同的執行續，所以如果用上述的方式在執行續之間頻繁的切換很浪費時間。DECQUEUE的主要作用就是使用QUEUE，設定QUEUE的容量是64，也就是最多可以掃描64個檔案，當然QUEUE裡面放的是檔名，不是檔案，節省不必要的空間浪費。而視窗一樣可以從QUEUE取出檔名，只要QUEUE不為空。

## 貳、標頭檔
```cpp=
#pragma once
#include <Windows.h>
#include <tchar.h>
#include <iostream>

#define DECQUEUE_SCANONLY

#ifndef DECQUEUE_SCANONLY
#include "../WannaTry/WanaDecryptor.h"
#endif

// queue裡面的最大檔案數量
#define MAXQUEUE 64  
// 初始狀態
#define IDC_DECQUEUE_NONE 0 
// STOP 跟 START 沒有用到
#define IDC_DECQUEUE_START 1
#define IDC_DECQUEUE_STOP 2  // 開始解密
// 當結束整個掃描後會傳送訊息給視窗
#define IDC_DECQUEUE_DONE 3  
// 執行續讀取檔案後用來傳遞訊息
#define IDC_DECQUEUE_DATA 4 
// 傳遞檔名和檔案屬性
struct _FILEINFO {
    TCHAR m_szName[MAX_PATH + 1];
    DWORD m_dwFileAttributes;
};

class DECQUEUE {
private:
    HANDLE m_hStopEvent;    // 事件
    HANDLE m_hFillCount = NULL;  // 號誌1
    HANDLE m_hEmptyCount = NULL; // 號誌2
    CRITICAL_SECTION m_CriticalSection;  // 臨界區域
    _FILEINFO m_aFileInfo[MAXQUEUE];  // 檔名的queue
    // queue head
    int m_iHead;
    // queue tail
    int m_iTail;
    // 數量計算
    int m_nCount;
    // 目前狀態
    int m_Status;
    // 接收指令
    int m_Command;

#ifndef DECQUEUE_SCANONLY
    PWanaDecryptor m_pDecryptor;
#endif
    HWND m_hWnd;
public:
    // 解密起始目錄
    LPCTSTR m_Start;  

    DECQUEUE(HWND);
    ~DECQUEUE();
    // 檔名入queue
    void SendData(LPCTSTR, DWORD);
    // 對話框從queue接收檔名
    BOOL RecvData(TCHAR*, PDWORD);
    // 掃描目錄
    BOOL Traverse(LPCTSTR, DWORD, DWORD);
    // 對話框停止傳送訊
    void Stop();
    // 檢查對話框是否傳來停止訊息
    BOOL CheckStopEvent();
};

typedef DECQUEUE* PDECQUEUE;

DWORD WINAPI DecQueueThread(LPVOID);
```
標頭檔有定義一個巨集DECQUEUE_SCANONLY，用來測試用，如果使用這個巨集就只會掃描目錄，不會進行解密。另外定義幾個狀態，IDC_DECQUEUE_NONE、IDC_DECQUEUE_START、IDC_DEQUEUE_STOP等，用來代表初始狀態、開始加密、暫停加密等等。 \_FILEINFO結構裡面有黨名與檔案屬性，放入檔案屬性是要判斷是否為目錄。<br>
在31~45行是在定義DEQUEUE會用到的變數，其中開頭三個HANDLE分別為事件與兩個號誌。特別提出來的原因是視窗與解密程式這樣的關系其實就是常見的"消費者生產者問題"，這邊是參考維基百科的解法，使用兩個號誌解決同步問題。另外有定義資料傳送、接收檔名、掃描目錄等函數。

## 參、類別實作
### 01. 建構子
```cpp=4
DECQUEUE::DECQUEUE(HWND hWnd = NULL)
{
    // 產生事件
    m_hStopEvent = CreateEvent(
        NULL,
        0,
        FALSE,
        NULL);
    // 產生號誌
    m_hFillCount = CreateSemaphore(
        NULL,
        0,
        MAXQUEUE,
        NULL);
    // 產生號誌
    m_hEmptyCount = CreateSemaphore(
        NULL,
        MAXQUEUE,
        MAXQUEUE,
        NULL);
    // 初始化臨界區域
    InitializeCriticalSection(&m_CriticalSection);
    m_iHead = 0;
    m_iTail = 0;
    m_nCount = 0;
    // 因為還沒開始掃描，所以放NONE
    m_Status = IDC_DECQUEUE_NONE;
    m_Command = IDC_DECQUEUE_NONE;

#ifndef DECQUEUE_SCANONLY
    m_pDecryptor = new WanaDecryptor();
#endif
    m_hWnd = hWnd;
}
```
建構函數基本上就只有初始化變數而已，然後產生停止事件、兩個Semaphore與臨界區域。

### 02. 解構子
```cpp=39
DECQUEUE::~DECQUEUE()
{
    if (m_hFillCount) {
        CloseHandle(m_hFillCount);
    }
    if (m_hEmptyCount) {
        CloseHandle(m_hEmptyCount);
    }
    if (m_hStopEvent) {
        CloseHandle(m_hStopEvent);
    }
    DeleteCriticalSection(&m_CriticalSection);
#ifndef DECQUEUE_SCANONLY
    delete m_pDecryptor;
#endif
}
```
解構函數關閉所有HANDLE與臨界區域。

### 03. 信息傳送
```cpp=56
void DECQUEUE::SendData(
    LPCTSTR pName,
    DWORD dwFileAttributes)
{
    // 等待queue空位
    WaitForSingleObject(m_hEmptyCount, INFINITE);
    // 進入臨界區域，一次只能一個執行續進入臨界區域
    EnterCriticalSection(&m_CriticalSection);

    m_aFileInfo[m_iHead].m_szName[0] = 0;
    m_aFileInfo[m_iHead].m_dwFileAttributes = 0;
    if (!pName) {  // 如果檔名為NULL代表掃描結束
        m_Status = IDC_DECQUEUE_DONE;
    }
    else if (m_nCount < MAXQUEUE) {
        m_Status = IDC_DECQUEUE_DATA;
        if (pName) {
            _tcscpy_s(m_aFileInfo[m_iHead].m_szName,
                MAX_PATH,
                pName);
        }
        m_aFileInfo[m_iHead].m_dwFileAttributes =
            dwFileAttributes;
        m_iHead = (m_iHead + 1) % MAXQUEUE;
        m_nCount++;
    }
    LeaveCriticalSection(&m_CriticalSection);
    ReleaseSemaphore(m_hFillCount, 1, NULL);
    if (m_hWnd) {
        SendMessage(m_hWnd, WM_USER, m_Status, NULL);
    }
}
```
首先要等待m_hEmptyCount信號才能進入CriticalSection。然後這邊比較特別的是使用環狀隊列，所以m_iHead的前進與m_iTail後退都要取MAXQUEUE的餘數。

### 04. 接收資料
```cpp=89
BOOL DECQUEUE::RecvData(
    TCHAR* pName,
    PDWORD pdwFileAttributes)
{
    WaitForSingleObject(m_hFillCount, INFINITE);
    EnterCriticalSection(&m_CriticalSection);
    BOOL bResult = TRUE;
    if (m_Status == IDC_DECQUEUE_DONE) {
        bResult = FALSE;
    }
    if (m_nCount > 0) {
        if (pName) {
            _tcscpy_s(pName,
                MAX_PATH,
                m_aFileInfo[m_iTail].m_szName);
        }
        if (pdwFileAttributes) {
            *pdwFileAttributes =
                m_aFileInfo[m_iTail].m_dwFileAttributes;
        }
        m_iTail = (m_iTail + 1) % MAXQUEUE;
        m_nCount--;
    }
    LeaveCriticalSection(&m_CriticalSection);
    ReleaseSemaphore(m_hEmptyCount, 1, NULL);
    return bResult;
}
```
接收資料也要等待號誌，只不過是m_hFillCount，內容跟傳送信號差不多，只不過一個是要倒退一個是前進而已。

### 05. 掃描目錄
```cpp=117
BOOL DECQUEUE::Traverse(
    LPCTSTR pPath,
    DWORD dwAttributes = 0,
    DWORD nLevel = 0)
{
    BOOL bResult = TRUE;
    if (CheckStopEvent()) {
        return FALSE;
    }
    if (!dwAttributes) {
        dwAttributes = GetFileAttributes(pPath);
        if (dwAttributes == INVALID_FILE_ATTRIBUTES) {
            return TRUE;
        }
    }
    if (!(dwAttributes & FILE_ATTRIBUTE_DIRECTORY)) {
#ifndef DECQUEUE_SCANONLY
        LPCTSTR pSuffix = _tcsrchr(pPath, _T('.'));
        if (pSuffix) {
            if (!_tcsicmp(pSuffix, WZIP_SUFFIX_CIPHER) ||
                !_tcsicmp(pSuffix, WZIP_SUFFIX_WRITESRC)) {
                m_pDecryptor->Decrypt(pPath);
                SendData(pPath, dwAttributes);
            }
            // 刪除解密站存檔
            else if (!_tcsicmp(pSuffix, WZIP_SUFFIX_TEMP)) {
                DeleteFile(pPath);
            }
        }
#else
        SendData(pPath, dwAttributes);
#endif
    }
    else {
        SendData(pPath, dwAttributes);
        TCHAR szFullPath[MAX_PATH + 1];
        WIN32_FIND_DATA FindFileData;
        _stprintf_s(szFullPath, _T("%s\\*.*"), pPath);
        HANDLE hFind = FindFirstFile(
            szFullPath,
            &FindFileData);
        if (INVALID_HANDLE_VALUE == hFind) {
            return TRUE;
        }
        do {
            if (!_tcscmp(FindFileData.cFileName, _T(".")) ||
                !_tcscmp(FindFileData.cFileName, _T(".."))) {
                continue;
            }
            _stprintf_s(szFullPath,
                _T("%s\\%s"),
                pPath,
                FindFileData.cFileName);
            bResult = Traverse(
                szFullPath,
                FindFileData.dwFileAttributes,
                nLevel + 1);
        } while (bResult &&
            FindNextFile(hFind, &FindFileData) != 0);
        FindClose(hFind);
    }
    if (nLevel <= 0) {
        SendData(NULL, 0);
    }
    return bResult;
}
```
掃描會用到檔案、目錄處理、檔案尋找的相關函數，[說明在這篇文章裡](https://hackmd.io/@hack-Ransomware/H1fIE1_Ku)。<br>
另外判斷那個路徑是目錄還是檔案可以用檔案屬性來看，如果是目錄，檔案屬性就會是FILE_ATTRIBUTE_DIRECTORY。

### 06. 停止相關函數
```cpp=184
void DECQUEUE::Stop()
{
    SetEvent(m_hStopEvent);
}

BOOL DECQUEUE::CheckStopEvent()
{
    DWORD retval = WaitForSingleObject(m_hStopEvent, 0);
    if (WAIT_OBJECT_0 == retval) {
        return TRUE;
    }
    return FALSE;
}
```
用來結束掃描用的，沒有特別的內容。

### 07. 解密程式
```cpp=198
#ifndef ENCRYPT_ROOT_PATH
#define ENCRYPT_ROOT_PATH _T("C:\\")
#endif

DWORD WINAPI DecQueueThread(
    _In_ LPVOID lpParameter
)
{
    PDECQUEUE pQueue = (PDECQUEUE)lpParameter;
    BOOL bResult = TRUE;
    if (pQueue->m_Start) {
        bResult = pQueue->Traverse(pQueue->m_Start);
    }
    else {
        TCHAR szRootPathName[16] = ENCRYPT_ROOT_PATH;
        for (INT DiskNO = 25; DiskNO >= 0; DiskNO--) {
            DWORD Drives = GetLogicalDrives();
            if ((Drives >> DiskNO) & 1) {
                szRootPathName[0] = DiskNO + 65;
                bResult = pQueue->Traverse(szRootPathName);
                if (!bResult) {
                    break;
                }
            }
        }
    }
    ExitThread(0);
}
```
這是解密的主程式，使用上面建構的函數來掃描目錄，另外有用到一個函數GetLogicalDrives()是用來取的磁碟機的。


### 08. main函數
```cpp=227
#ifdef DECQUEUE_SCANONLY
int main()
{
    PDECQUEUE pQueue = new DECQUEUE();
    HANDLE hThread;
    TCHAR szFileName[MAX_PATH + 1];
    DWORD dwAttributes;
    hThread = CreateThread(NULL, 0, DecQueueThread, pQueue, 0, NULL);
    int i = 0;
    while (TRUE) {
        if (!pQueue->RecvData(szFileName, &dwAttributes)) {
            break;
        }
        _tprintf(_T("%d Recv %s\n"), i, szFileName);
        i++;
        if (i >= 100) {
            pQueue->Stop();
        }
    }
    return 0;
}
#endif
```
main函數建立一個執行緒，指定給函數DecQueueThread。然後使用while loop開始接收檔案。
