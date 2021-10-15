---
title : 05_RC4加密-使用BCrypt
---


# BCrypt函式庫

by WL, Liu & BG, Tasi
Date : 2021-10-8

---

## 壹、基本函式
|    作用    |              函式            |          功能          |
|:----------:|:---------------------------:|:----------------------:|
| 開始與準備 | BCryptOpenAlgorithmProvider | 載入並初始化演算法題供者 |
| 開始與準備 | BCryptGenerateSymmetricKey  |     產生對稱式金鑰     |
| 加密、解密 |         BCryptEncrypt         |          加密          |
| 加密、解密 |         BCryptDecrypt         |         解密          |
| 結束關閉 |         BCryptDestroyKey         |         銷毀金鑰          |
| 結束關閉 |         BCryptCloseAlgorithmProvider         |         關閉演算法提供者          |

## 貳、函數介紹

### 01. 開啟演算法提供者-BCryptOpenAlgorithmProvider

* 在這裡我們是選用微軟提供的演算法提供者，也可指定其他特定提供者的演算法。
* 在MSDN說明文件上看到BCryptOpenAlgorithmProvider的宣告使用四個參數:
    * **phAlgorithm** => BCryptOpenAlgorithmProvider()呼叫成功時，會將資料寫入phAlgorithm，並指向接收 CNG 提供程序句柄的BCRYPT_ALG_HANDLE變量的指針。使用完畢後，通過BCryptCloseAlgorithmProvider()函數來釋放它。
    * **pszAlgId** => 向以空字符結尾的 Unicode 字串的指針，該字符串用來指定所求的加密算法。以下列舉其中三個:
        |        Unicode 字串        |       演算法        |
        |:--------------------------:|:-------------------:|
        | BCRYPT_RC4_ALGORITHML"RC4" |      RC4演算法      |
        | BCRYPT_RC4_ALGORITHML"RSA" |  RSA公開金鑰演算法  |
        | BCRYPT_RC4_ALGORITHML"AES" | AES對稱式加密演算法 |
        
    * **pszImplementation** => 指向以空字符結尾的 Unicode 字串的指針，該字符串標識要加載的特定提供程序。可以為NULL。如果此參數為NULL，則將加載指定算法的默認提供程序。以下為目前提供者:
        | Unicode 字串                          | 提供者                                                                 |
        | ------------------------------------- |:---------------------------------------------------------------------- |
        | L"Microsoft Primitive Provider"       | Identifies the basic Microsoft CNG provider                            |
        | L"Microsoft Platform Crypto Provider" | Identifies the TPM key storage provider that is provided by Microsoft. |
        
    * **dwFlags** => 旗標，是用來修改函數的，這可以是零或以下值中的一個或多個的組合:
        | 旗標 | 意義 |
        | ---- | ---- |
        | BCRYPT_ALG_HANDLE_HMAC_FLAG     |  提供程序將執行 Hash-Based Message Authentication Code (HMAC)演算法，此標誌僅由散列算法提供程序使用。    |
        |   BCRYPT_PROV_DISPATCH   |   指定此標誌時，在釋放所有相關對象之前不得關閉返回的句柄。   |
        | BCRYPT_HASH_REUSABLE_FLAG | 創建一個可重用的散列對象。該對象可以在調用BCryptFinishHash後立即用於新的散列操作。 |
        
* BCrypt所有的函數，回傳值都是NTSTATUS這個資料型別，而NTSTATUS定義在ntdef.h中，以下是回傳結果:
    | 回傳值 | 描述 |
    | ------ | ---- |
    |    STATUS_SUCCESS    |   成功   |
    |    STATUS_NOT_FOUND    |   找不到指定算法 ID 的提供者。   |
    |    STATUS_INVALID_PARAMETER    |  參數無效    |
    |    STATUS_NO_MEMORY   | 記憶題配值出現問題 |
    
    可先判定回傳值是否為STATUS_SUCCESS，若不是，在近一步了解錯誤原因。以上回傳值定義在ntstatus.h引入檔。
    
### 02. 產生對稱式金鑰-BCryptGenerateSymmetricKey
* BCryptGenerateSymmetricKey總共有七個參數，在某些加密方法並未完全用到，填入0或NULL即可。以下分別介紹七個參數:
    * **hAlgorithm** => 指向BCryptOpenAlgorithmProvider()函數創建的算法提供程序的結構。創建提供程序時指定的算法必須支持對稱密鑰加密。
    * **phKey** => 一個指向接收密鑰BCRYPT_KEY_HANDLE的指針。先準備BCRYPT_KEY_HANDLE以用來將其地址傳給BCryptGenerateSymmetricKey()，呼叫成功後會取得金鑰的HANDLE。當不再需要時，必須通過BCryptDestroyKey()函數來釋放它。
    * **pbKeyObject** => 指向接收密鑰對象的緩衝區的指針。可以通過調用BCryptGetProperty()函數獲取BCRYPT_OBJECT_LENGTH屬性來獲取此緩衝區所需的空間大小。
    * **cbKeyObject** => pbKeyObject緩衝區的大小（資料型態為byte）。必須是BCryptGetProperty()取得BCRYPT_OBJECT_LENGTH的值。
    * **pbSecret** => 指向緩衝區的指針，緩衝區從中創建密鑰對象的密鑰。cbSecret參數包含該緩衝區的大小(資料型態為byte)。這通常是密碼或其他一些可複製數據的散，資料型態為byte。若傳入的數據超過了目標key大小，數據超出的部分會忽略。
    * **cbSecret** => 密碼的長度，也就是pbSecret的長度（資料型態為byte）。
    * **dwFlags** => 目前保留未使用，填入0即可。
* 這函數回傳結果如下:
    | 回傳值 | 描述 |
    | ------ | ---- |
    |    STATUS_SUCCESS    |   成功   |
    |    STATUS_BUFFER_TOO_SMALL    |   cbKeyObject的值太小   |
    |    STATUS_INVALID_HANDLE    |  提供者之HANDLE無效    |
    |    STATUS_INVALID_PARAMETER   | 參數無效 |
### 03. 加密與解密-BCryptEncrypt & BCryptDecrypt
* 類似於BCryptGenerateSymmetricKey，若沒用到直接給0或NULL即可。以下為十個BCryptEncrypt的參數:
    * **hKey** => 用於加密數據的密鑰，從密鑰創建函數之一獲得的，例如: BCryptGenerateSymmetricKey()取得的對稱式金鑰、BCryptGenerateKeyPair()產生的非對稱式金鑰、BCryptImportKey()匯入(import)取得的金鑰。
    * **pbInput** => 被加密的明文的地址，資料型態為byte。
    * **cbInput** => 被加密的明文pbInput的大小，以byte為單位。
    * **pPaddingInfo** => 指向包含填充信息的結構的指針，關於Padding的資訊，此參數僅用於非對稱密鑰和經過身份驗證的加密模式。如果使用經過身份驗證的加密模式，則此參數必須指向BCRYPT_AUTHENTICATED_CIPHER_MODE_INFO結構，如果使用非對稱密鑰，則此參數指向的結構類型由dwFlags參數的值決定。反之，參數必須設置為NULL。
    * **pbIV** => 要在加密期間使用的初始化向量的緩衝區的地址，可以通過調用BCryptGetProperty()參數pszProperty為BCRYPT_BLOCK_LENGTH屬性來獲取所需的大小。
    * **cbIV** => 初始向量的大小，以byte為單位，也就是pbIV緩衝區的大小，必須是BCryptGetProperty()參數pszProperty為BCRYPT_BLOCK_LENGTH取得的值。
    * **pbOutput** => 接收此函數產生的密文的緩衝區地址，cbOutput參數包含該緩衝區的大小。如果此參數為NULL，則BCryptEncrypt函數計算pbInput參數中傳遞的數據的密文所需的大小。在這種情況下，pcbResult參數指向的位置包含此大小，並且函數返回STATUS_SUCCESS。
    * **cbOutput** => pbOutput緩衝區的大小，以byte為單位。
    * **pcbResult** => 指向ULONG變量的指針，該變量接收存入到pbOutput緩衝區的byte數。如果pbOutput為NULL，則接收密文所需的大小（以byte為單位）。
    * **dwFlags** => 一組修改此函數行為的標誌。允許的標誌集取決於hKey參數指定的密鑰類型。如果密鑰是對稱密鑰，則這可以是零或以下值。
    

        | 值            | 意義                                                                                                                                                                                                              |
        | -------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
        | BCRYPT_BLOCK_PADDING | 允許加密算法將數據填充到下一個塊大小。如果未指定此標誌，則cbInput參數中指定的明文大小必須是算法塊大小的倍數。可以通過調用BCryptGetProperty()函數獲取密鑰的BCRYPT_BLOCK_LENGTH屬性來獲取塊大小。這將為算法提供塊的大小。     |
    
        若密鑰是非對稱密鑰，則可以是以下值之一。
        | 值 | 意義 |
        | -------- | -------- |
        | BCRYPT_PAD_NONE         |  不用任何填充，pPaddingInfo不使用此參數，cbInput參數中指定的明文大小必須是算法塊大小的倍數。        |
        |  BCRYPT_PAD_OAEP        |  使用最佳非對稱加密填充 (OAEP) 方案，所述pPaddingInfo參數是一個指向BCRYPT_OAEP_PADDING_INFO結構。        |
        | BCRYPT_PAD_PKCS1     | 數據將填充一個隨機數以四捨五入大小，pPaddingInfo不使用此參數。     |

* 以下為可能出現的返回結果與其意義:

    | 返回值 | 描述 |
    | -------- | -------- |
    | STATUS_SUCCESS         |  成功        |
    | STATUS_BUFFER_TOO_SMALL         |  cbOutput參數指定的大小不足以容納密文        |
    | STATUS_INVALID_BUFFER_SIZE         |  所述cbInput參數不是區塊大小的整數倍        |
    | STATUS_INVALID_HANDLE         |  hKey有問題        |
    | STATUS_INVALID_PARAMETER         |  參數出現錯誤        |
    | STATUS_NOT_SUPPORTED     | 這演算法不支持加密     |
* BCryptEncrypt與BCryptDecrypt參數幾乎一模一樣，使用上也一樣，僅效果不同，一作為加密一作為解密，在此不多做贅述。
### 04. 銷毀金鑰-BCryptDestroyKey
* 加密或解密結束時，一定要銷毀金鑰，否則可從記憶體中取出解密金鑰，從而將檔案成功破解。
* 僅有一個參數**hkey**，即輸入要刪除之金鑰即可。
* 回傳結果如下:

    | 回傳值 | 意義 |
    | ------ | ---- |
    | STATUS_SUCCESS       |  成功    |
    | STATUS_INVALID_HANDLE   | hkey有問題 |

### 05. 關閉演算者提供者-BCryptCloseAlgorithmProvider
* 這是使用BCrypt的最後一步，用於關閉提供者，以下輸入之參數:
    * hAlgorithm => 關閉算法提供者，算法提供者由BCryptCloseAlgorithmProvider()產生。
    * dwFlags => 一組修改此函數行為的標誌，目前沒有為此函數定義標誌。
* 回傳結果如下:

    | 回傳值 | 意義 |
    | -------- | -------- |
    | STATUS_SUCCESS         |  成功        |
    | STATUS_INVALID_HANDLE     | hAlgorithm參數中的HANDLE有問題     |

## 參、RC4加密實作(使用BCrypt)

```clike=
#include <stdio.h>
#include <string.h>
#include <Windows.h>
#include <bcrypt.h>
#include <tchar.h>
#pragma comment(lib, "bcrypt")


#ifndef NT_SUCCESS
#define NT_SUCCESS(Status) ((NTSTATUS)(Status) >= 0)
#endif

#ifdef STATUS_UNSUCCESSFUL
#define STATUS_UNSUCCESSFUL 0xC0000001
#endif

#define CHECHERR(n, m){if(!NT_SUCCESS(n)){printf("Error: %s\n, m"); exit(1);}}

int  rc4_sbox_1(unsigned char sbox[256]) {
	for (int i = 0; i < 256; i++) {
		sbox[i] = i;
	}
	return 0;
}

int rc4_sbox_2(unsigned char sbox[256], const unsigned char* password) {
	int password_len = strlen((char*)password);
	for (int i = 0, j = 0; i < 256; i++) {
		j = (j + sbox[i] + password[i % password_len]) % 256;
		int t = sbox[i];
		sbox[i] = sbox[j];
		sbox[j] = t;
	}
	return 0;
}

int rc4_encrypt(unsigned char sbox[256], unsigned char* data, int size, unsigned char* output) {
	for (int i = 0, j = 0, k, p = 0; p < size; p++) {
		i = (i + 1) % 256;
		j = (j + sbox[i]) % 256;
		int t = sbox[i];
		sbox[i] = sbox[j];
		sbox[i] = sbox[j];
		sbox[j] = t;
		k = (sbox[i] + sbox[j]) % 256;
		output[p] = data[p] ^ sbox[k];
	}
	return 0;
}

ULONG hexdump(PUCHAR data, ULONG size) {
	ULONG nResult = 0;
	for (ULONG i = 0; i < size; i += 16) {
		nResult += printf("%08X |", i);
		for (ULONG j = 0; j < 16; j++) {
			if (i + j < size) {
				nResult = printf(" %02X", data[i + j]);
			}
			else {
				nResult += printf("  ");
			}
			if ((i + 1) % 8 == 0) {
				nResult += printf(" ");
			}
		}
		nResult += printf("|");
		for (ULONG j = 0; j < 16; j++) {
			if (i + j < size) {
				UCHAR k = data[i + j];
				UCHAR c = k < 32 || k>127 ? '.' : k;
				nResult += printf("%c", c);
			}
			else {
				nResult += printf(" ");
			}
		}
		nResult += printf("\n");
	}
	return nResult;
}

int main(void) {
	unsigned char sbox[256];
	unsigned char abText[1024] = "this is a test";
	ULONG len = strlen((char*)abText) + 1;
	unsigned char abCipher[1024];
	unsigned char* pnPlain;
	ULONG cbPlain;
	printf("Text: \n");
	hexdump(abText, len);
    printf("RC4 Encrypted:\n");
	rc4_sbox_1(sbox);
	rc4_sbox_2(sbox, (unsigned char*)"password");
	rc4_encrypt(sbox, abText, len, abCipher);
	hexdump(abCipher, len);
	printf("decrypt by BCrypt: \n");
    rc4_sbox_1(sbox);
	rc4_sbox_2(sbox, (unsigned char*)"password");
	rc4_encrypt(sbox, abCipher, len, abText);
	hexdump(abText, len);
	NTSTATUS status;
	BCRYPT_HANDLE hProv;
	BCRYPT_KEY_HANDLE hkey;
	status = BCryptOpenAlgorithmProvider(
		&hProv,
		BCRYPT_RC4_ALGORITHM,
		NULL, 0);
}
```
* 19~24行 S-Box初始化第一階段(將S-Box裡所有值初始化，將長度256的字元陣列填上0x00~0xFF)
* 26~34行 S-Box初始化第二階段(初始化和密碼做運算，期間每次會有兩個位置裡的值交換，增加混亂程度)
* 37~48行 正式加密(準備好S-Box後才可加密,再加密過程中，S-Box的內容不停地在改變，現今破解方法是利用統計來猜測的)(加密的部分是透過特定計算，取出S-Box中某位置的值做XOR運算，同時還會將S-Box裡兩個位置的值交換，加密過程仍在改變S-Box的內容，增加破解難度)
* 82~91行 為整個加密的流程
* 92~96行 為解密的流程
* 97~107行 為加密與解密完後摧毀金鑰與關閉演算法提供者
