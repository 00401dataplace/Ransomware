---
title : 03_Windows Data Type
---
# Windows Data Type
By Cheng-Yen, Tsai
Date : 2021-05-29

---
## 簡介
針對Windows 32bit api，microsoft有定義很多自己的資料型態，而這街資料型態與早期32位元程式有很大的關係，例如長指標、寬字元等等，在現行的C/C++內比較少用到，但是因為windows api很多都是32位元，所以如果不了解這些資料型態的命名以及它代表的意義，在使用相關函數會遇到看不懂參數的問題。 這些資料型態都有一定的命名規則，例如Ｌ代表長，Ｃ代表常數等，所以如果把這個命名規則看懂就可以不用記住全部的資料型態，利用IDE的提示功能，就可以把對應的參數放進去。
## 字元型態
字元型態的規則就是C、T、U、W，分別代表常數、UINCODE、unsigned、wchar。
* CHAR => 8-bit, typedef char CHAR;
* CCHAR => 8-bit, typedwf char CCHAR;
* TCHAR => 16-bit, #ifdef UNICODE typedef WCHAR TCHAR;            8-bit,  #else typedef char TCHAR;
* UCHAR => 8-bit, typedef unsigned char UCHAR;
* WCHAR => 16-bit, typedef wchar_t WCHAR;
---
### 整數型態
整數型態就比較單純，INT後面接他的位元大小，另外U就是unsigned的意思。
* INT => 32-bit, typedef int INT;
* INT8 => 8-bit, typedef singed char INT8;
* INT16 => 16-bit, typedef signed short INT16;
* INT32 => 32-bit, typedef signed int INT32;
* INT64 => 64-bit, typedef signed _int64 INT64;
* UINT => 32-bit, typedef unsigned int UINT;
* UINT8 => 8-bit, typedef unsigned char UINT8;
* UINT16 => 16-bit, typedef unsigned short;
* UINT32 => 32-bit, typedef unsigned int UINT32;
* UINT64 => 64-bit, typedef unsigned _int64 UINT64;
---

### 整數型態-DWORD
DWORD的完整名稱是Double word，是word的兩倍，命名規則比較單純。
* DWORD => 32-bit, typedef unsigned long DWORD;
* DWORDLONG => 64-bit, typedef unsigned _int64 DWORDLONG;
* DWORD32 => 32-bit, typedef unsigned int DWORD32;
* DWORD64 => 64-bit, typedef unsigned _int64 DWORD64;
---

### 短整數型態
短整數包含BYTE、WORD、SHORT，命名規則一樣是U與T。
* BYTE => 8-bit, typedef unsigned char BYTE;
* TBYTE => 16-bit, #ifdef UNICODE typedef WCHAR TBYTE; 
*          8-bit,  #else typedef unsigned char TBYTE;
* WORD => 16-bit, typedef unsigned short WORD;
* SHORT => 16-bit, typedef short SHORT;
* USHORT => 16-bit, typedef unsigned short USHORT;
---

### 指標型態-長指標LP
長指標是現在比較少見，但是win32 api很多函數都有用到，定義也都是用現在常用的型態去定義。命名規則比較複雜，但是從下面的說明就可以很清楚的比較出來。
* LPCSTR => 8-bit, typedef _nullterminated CONST CHAR *LPCSTR;
* LPCTSTR => 16-bit, #ifdef UNICODE typedef LPCWSTR LPCTSTR;    8-bit,  #else typedef typedef LPCSTR LPCTSTR;
* LPCWSTR => 16-bit, typedef CONST WCHAR *LPCWSTR;
* LPSTR => 8-bit, typedef CHAR *LPSTR;
* LPTSTR => 16-bit, #ifdef UNICODE typedef LPWSTR LPTSTR;
          8-bit,  #else typedef LPSTR LPTSTR;
* LPWSTR => 16-bit, typedef WCHAR *LPWSTR;
* LPVOID => auto, typedef void *LPVOID;
---

### 字元與字串指標型態
字元與字串的指標型態是用上面的字元型態加上指標來定義，所以也是相對簡單的。
* PBYTE => typedef BYTE *PBTE;
* PCHAR => typedef CHAR *PCHAR;
* PTCHAR => typedef TCHAR *PTCHAR;
* PWCHAR => typedef WCHAR *PWCHAR;
* PCSTR => typedef CONST CHAR *PCSTR;
* PCTSTR => #ifdef UNICODE typedef LPCWSTR PCTSTR;
           #else typedef LPCSTR PCTSTR;
* PCWSTR => typedef CONST WCHAR PCWSTR;
* PSTR => typedef CHAR *PSTR;
* PTSTR => #ifdef UNICODE typedef LPWSTR PTSTR;
          #else typedef LPSTR PTSTR;
* PWSTR => typedef WCHAR *PWSTR;

### 結語
windows api還有很多特殊的資料型態，例如內核物件的handle，但是也是用int定義的，所以在了解這個之前要把這些基本的搞清楚。在深入win32 api後會遇到很多沒看過的型態，但是這些都是基本的型態用typedef重新定義的，所以瞭解基本的命名規則是必要的。