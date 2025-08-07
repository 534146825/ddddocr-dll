# ddddocr-dll
通过原rust 链接:https://github.com/86maid/ddddocr的ddddocr打包的dll文件，给golang识别调用。目前只支持字母数字的，的验证码。
调用方式如下：
package main

/*
#include <stdlib.h>
*/
import "C"
import (
	"encoding/base64"
	"fmt"
	"os"
	"syscall"
	"unsafe"
)

func CallRustIdentify(imgData []byte) (string, error) {
	// 1. 加载DLL
	dll, err := syscall.LoadDLL("ddddocr.dll")
	if err != nil {
		return "", fmt.Errorf("加载DLL失败: %v", err)
	}
	defer dll.Release()

	// 2. 查找函数
	identify, err := dll.FindProc("identify")
	if err != nil {
		return "", fmt.Errorf("找不到identify函数: %v", err)
	}

	freeString, err := dll.FindProc("free_string")
	if err != nil {
		return "", fmt.Errorf("找不到free_string函数: %v", err)
	}

	// 3. 准备Base64参数
	base64Str := base64.StdEncoding.EncodeToString(imgData)
	cstr := C.CString(base64Str)
	defer C.free(unsafe.Pointer(cstr))

	// 4. 调用Rust函数
	ret, _, err := syscall.Syscall(identify.Addr(), 1,
		uintptr(unsafe.Pointer(cstr)),
		0, 0)
	if ret == 0 {
		return "", fmt.Errorf("rust返回了空指针")
	}

	// 5. 转换结果并释放内存
	defer syscall.Syscall(freeString.Addr(), 1, ret, 0, 0)
	result := C.GoString((*C.char)(unsafe.Pointer(ret)))

	return result, nil
}

func main() {
	// 读取图片
	imgBytes, err := os.ReadFile("test.png")
	if err != nil {
		fmt.Printf("读取图片失败: %v\n", err)
		return
	}

	// 调用OCR识别
	result, err := CallRustIdentify(imgBytes)
	if err != nil {
		fmt.Printf("识别失败: %v\n", err)
		return
	}

	fmt.Println("识别结果:", result)
}

