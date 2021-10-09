---
title : 06_windows視窗
---

# Windows視窗
By Cheng-Yen, Tsai & Wei-Liang, Liu
Date : 2021-09-24

---
## 壹、前言
介面程式是勒索程式重要的一環，視窗可作為與勒索者之橋梁，其中的每個文字、按鈕、編輯框、圖片等等，都是需要透過程式設計的編寫。雖然稱為視窗，但卻不是只有視窗的功能，例如木馬程式的鍵盤側錄器，或是畫面上看到的計時器、外部網站等等都是要用到視窗程式提供的函數。
![](https://i.imgur.com/sLxLWwA.jpg)


---
## 貳、Windows視窗程式基礎架構
不同於一般主控台應用程式，windows程式的架構我認為較複雜。
1. 首先是程式進入點，傳統C/C++程式進入點為main()，而windows的視窗程式進入點為wWinMain()，在visual studio 2019可以直接建立windws視窗程式專案。這裡可以放相關的運算程式，視窗執行會一起進行運算。
```cpp=
int APIENTRY wWinMain(_In_ HINSTANCE hInstance,
                     _In_opt_ HINSTANCE hPrevInstance,
                     _In_ LPWSTR    lpCmdLine,
                     _In_ int       nCmdShow)
{
    UNREFERENCED_PARAMETER(hPrevInstance);
    UNREFERENCED_PARAMETER(lpCmdLine);

    // TODO: 在此放置程式碼。

    // 將全域字串初始化
    LoadStringW(hInstance, IDS_APP_TITLE, szTitle, MAX_LOADSTRING);
    LoadStringW(hInstance, IDC_WINDOWSPROJECT, szWindowClass, MAX_LOADSTRING);
    MyRegisterClass(hInstance);

    // 執行應用程式初始化:
    if (!InitInstance (hInstance, nCmdShow))
    {
        return FALSE;
    }

    HACCEL hAccelTable = LoadAccelerators(hInstance, MAKEINTRESOURCE(IDC_WINDOWSPROJECT));

    MSG msg;

    // 主訊息迴圈:
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
2. 專案建立後會看到wWinMain()，裡面可以寫一些程式，而這些程式會在視窗出現之前執行。在主程式下面有MyRegisterClass，中文翻譯是註冊視窗類別，但是我覺得有點難懂，其實就是把這個window的外觀、名稱等等設定好，基本上不會去動這個函數。
```cpp=
ATOM MyRegisterClass(HINSTANCE hInstance)
{
    WNDCLASSEXW wcex;

    wcex.cbSize = sizeof(WNDCLASSEX);

    wcex.style          = CS_HREDRAW | CS_VREDRAW;
    wcex.lpfnWndProc    = WndProc;
    wcex.cbClsExtra     = 0;
    wcex.cbWndExtra     = 0;
    wcex.hInstance      = hInstance;
    wcex.hIcon          = LoadIcon(hInstance, MAKEINTRESOURCE(IDI_WINDOWSPROJECT));
    wcex.hCursor        = LoadCursor(nullptr, IDC_ARROW);
    wcex.hbrBackground  = (HBRUSH)(COLOR_WINDOW+1);
    wcex.lpszMenuName   = MAKEINTRESOURCEW(IDC_WINDOWSPROJECT);
    wcex.lpszClassName  = szWindowClass;
    wcex.hIconSm        = LoadIcon(wcex.hInstance, MAKEINTRESOURCE(IDI_SMALL));

    return RegisterClassExW(&wcex);
}
```
3. 在設定好視窗後，下面是InitInstance()，用來產生主視窗，，並顯示我們自己更動好的結果。
```cpp=
BOOL InitInstance(HINSTANCE hInstance, int nCmdShow)
{
   hInst = hInstance; // 將執行個體控制代碼儲存在全域變數中

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
```
4. WndProc()是主要更動的地方，而這裡是一種事件導向的架構。一般的程式都是從主程式、副程式這樣一直執行下去，但是視窗程式不行，因為要考慮到鍵盤的輸入，滑鼠點擊滾動等等動作，所以要使用事件導向的方式進行。
事件導向的函數通常是Call Back型態，是指能藉由參數通往另一個函式的函式，所以裡面可以看到是使用switch來判斷要跳往哪個動作。
```cpp=
LRESULT CALLBACK WndProc(HWND hWnd, UINT message, WPARAM wParam, LPARAM lParam)
{
    switch (message)
    {
    case WM_COMMAND:
        {
            int wmId = LOWORD(wParam);
            // 剖析功能表選取項目:
            switch (wmId)
            {
            case IDM_ABOUT:
                DialogBox(hInst, MAKEINTRESOURCE(IDD_ABOUTBOX), hWnd, About);
                break;
            case IDM_EXIT:
                DestroyWindow(hWnd);
                break;
            default:
                return DefWindowProc(hWnd, message, wParam, lParam);
            }
        }
        break;
    case WM_PAINT:
        {
            PAINTSTRUCT ps;
            HDC hdc = BeginPaint(hWnd, &ps);
            // TODO: 在此新增任何使用 hdc 的繪圖程式碼...
            EndPaint(hWnd, &ps);
        }
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

## 視窗相關函數
1. **新增視窗** **:** CreateWindow(), CreateWindowEx()

    針對視窗創建有兩種方式，一種是CreateWindow()，是創建主要視窗的函數，另一種是CreateWindowEx()，是用來創建主視窗內物件的函數，可以理解成在主視窗內，像是按鈕、下拉選單等等都是一個子視窗，主視窗的呼叫就是透過CreateWindow()產生出來的。CreateWindowEx()僅比CreateWindow()多一個參數，其第一個參數表示擴充樣式，其餘皆相同。
    CreateWindowEx()的參數如下:
    * **dwExStyle** => 擴充視窗樣式，通常不常使用，可填入0。
    * **lpClassName** => 窗口的類別名稱，以0字元為結尾的字串，也可是RegisterClass或RegisterClassEx產生的ATOM。
    * **lpWindowName** => 視窗上的標題，是以0為結尾的字串。
    * **dwStyle** => window之樣式選擇。常用到的有：
        | 名稱 | 解釋 |代碼
        | -------- | -------- | -------- |
        | WS_VSCROLL | 增加後右側會有scroll bar     | 0x00200000L     |
        | WS_VISIBLE | 這個物件被初始化成可以看見的 | 0x10000000L |
        | WS_TABSTOP | 加入這個style可以讓用戶按下tab鍵後跳至另一個物件 | 0x00010000L |
        | WS_CHILD | 代表這個視窗是子視窗 | 0x40000000L |
    * **x** => 視窗的x座標，或是輸入CW_USEDEFAULT設為預設值。
    * **y** => 視窗的y座標，或是輸入CW_USEDEFAULT設為預設值。
    * **Width** => 視窗的寬度，或是輸入CW_USEDEFAULT設為預設值。
    * **nHeight** => 視窗的長度，或是輸入CW_USEDEFAULT設為預設值。
    * **hWndParent** => 產生視窗程式之視窗handle。
    * **hMenu** => 可為選單handle，或是用以指定的子視窗標識(為一整數值)，該值由視窗樣式來決定。如果是menu，可放入NULL。
    * **Instance** => 產生視窗程序的handle，通常放入全域變數hInst。
    * **ipParam** => 從CreateWindow產生視窗時，產生WM_CREATE訊息，而參數指向CREATESTRUCT結構。若產生的是MDI客戶視窗，則指向CLIENTCREATESTRUCT結構。
    
    回傳值為新視窗的handle，失敗則回傳NULL，可由GetLastError取得錯誤碼。
2. 傳送訊息：SendMessage()
    許多元件裡都需有文字，像是按鈕、編輯框中都需要文字，決定傳達文字與字型就是要用到SendMessage()。以下四個為該函示之參數:
    * **hWnd** => 接收訊息視窗的handle，若放入HWND_BROADCAST，則表示廣播給所有最頂端的視窗。
    * **Msg** => 要傳送之訊息，可分為系統定義與應用程式定義訊息。系統定義表示window本身的訊息，例如WM_PAINT、progress bar、combobox等，若要設定字型，要填入WM_SETFONT。
    * **wParam** => 訊息所附加的資訊。已設定字型來說，則放CreateFont產生的handle，或是NULL代表預設字型
    * **IParam** => 訊息所附加的資訊。