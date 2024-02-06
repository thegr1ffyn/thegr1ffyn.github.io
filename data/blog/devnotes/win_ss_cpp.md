---
title: Taking Screenshots using Windows API in C++
date: '2023-06-27'
tags: ['dev-notes', 'windows-api', 'c++', 'screenshot', 'bmp']
draft: false
summary: Utilizing Windows API in C++ to take screenshot and store it in file and in memory. The output is in BMP format and can be stored in either a file or in memory.
---

## Introduction

In this post, we'll be taking a look at how we can take screenshots using Windows API in C++. The output will be in BMP format and can be stored in either a file or in memory. Firstly, let me breakdown how I approached this problem and what I did to solve it. Firstly, there are a few things that I had defined on how I can apporach this:

1. Finding the total size of the screen.
2. Taking the screenshot.
3. Saving the image to a file or storing it in memory.

Now, let's get started.

## Finding the total size of the screen

To find the total size of the screen, we will be using the `GetSystemMetrics` function. This function takes in a parameter which is the `SM_CXSCREEN` and `SM_CYSCREEN` which are the width and height of the screen respectively. The function returns the width and height of the screen in pixels. According to [MSDN](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-getsystemmetrics)

> Retrieves the specified system metric or system configuration setting. Note that all dimensions retrieved by GetSystemMetrics are in pixels.

```cpp:GetSystemMetrics
int GetSystemMetrics(
  [in] int nIndex
);
```

Now, the parameter can be an `int` which is the index of the system metric or system configuration setting to be retrieved. There are a total of `107` possible values that can be passed to the function. The complete list can be found [here](https://docs.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-getsystemmetrics#parameters). For our purpose, we will be using the `SM_CXSCREEN` and `SM_CYSCREEN` which are the width and height of the screen respectively. The function returns the width and height of the screen in pixels.

| Index | Value | Description |
| --- | --- | --- |
| 0 | SM_CXSCREEN | The width of the screen of the primary display monitor, in pixels. |
| 1 | SM_CYSCREEN | The height of the screen of the primary display monitor, in pixels. |

Now, let's a simple program that will print the width and height of the screen.

```cpp:driver.cpp
#include <iostream>
#include <windows.h>

int main() {
    
    int sWidth  = GetSystemMetrics(SM_CXSCREEN);
    int sHeight = GetSystemMetrics(SM_CYSCREEN);

    printf("Width : %d\n", sWidth);
    printf("Height: %d\n", sHeight);

    return 0;
}
```

The output will be something like this:

```bash
Width : 1920
Height: 1080
```

## Taking the screenshot

This portion will further be divided into two parts:

1. Creating a device context.
2. Creating a bitmap.

### Creating a device context

Now, what is a device context? According to [MSDN](https://docs.microsoft.com/en-us/windows/win32/gdi/device-contexts)

> A device context is a structure that defines a set of graphic objects and their associated attributes, as well as the graphic modes that affect output. The graphic objects include a pen for line drawing, a brush for painting and filling, a bitmap for copying or scrolling parts of the screen, a palette for defining the set of available colors, a region for clipping and other operations, and a path for painting and drawing operations.

Now, we will be using the `GetDC` function to create a device context. This function takes in a parameter which is the `hWnd` which is a handle to the window whose device context is to be retrieved. If this value is `NULL`, `GetDC` retrieves the device context for the entire screen. The function returns a handle to a device context for the specified window or `NULL` if no device context exists for the window.

```cpp:GetDC
HDC GetDC(
  [in] HWND hWnd
);
```

Now, we know the usage is fairly simple, we will not provide any argument to the function and it will return a handle to the device context for the entire screen. The function usage will be as follows:

```cpp:driver.cpp
HDC hDC = GetDC(NULL);
```

### Creating a bitmap

Now, what is a bitmap? According to [MSDN](https://docs.microsoft.com/en-us/windows/win32/gdi/bitmaps)

> A bitmap is a type of memory organization or image file format used to store digital images. The term bitmap comes from the computer programming terminology, meaning just a map of bits, a spatially mapped array of bits. Now, along with the device context, we will be creating a bitmap using the `CreateCompatibleBitmap` function. This function takes in three parameters which are the `hDC` which is a handle to the device context, `nWidth` which is the width of the bitmap in pixels, and `nHeight` which is the height of the bitmap in pixels. The function returns a handle to the compatible bitmap if the function succeeds or `NULL` if the function fails.

```cpp:CreateCompatibleBitmap
HBITMAP CreateCompatibleBitmap(
  [in] HDC hdc,
  [in] int cx,
  [in] int cy
);
```

Now, we know the usage is fairly simple, we will provide the handle to the device context, the width and height of the screen. The function usage will be as follows:

```cpp:driver.cpp
HBITMAP hBitmap = CreateCompatibleBitmap(hDC, sWidth, sHeight);
```

However, before createing a bitmap, we need to create a `Compatible Device Context`. Now, what is a compatible device context? According to [MSDN](https://docs.microsoft.com/en-us/windows/win32/gdi/compatible-device-contexts)

> A compatible device context is a device context that has the same type, resolution, color format, and so on as the device context referenced by the hDC parameter. The newly created compatible device context is only a device context in memory. An application cannot select any objects into it, nor can it draw to the screen with it. To free the compatible device context, call the DeleteDC function.

Now, we will be using the `CreateCompatibleDC` function to create a compatible device context. This function takes in a parameter which is the `hDC` which is a handle to the device context to be copied. The function returns a handle to the compatible device context if the function succeeds or `NULL` if the function fails.

```cpp:CreateCompatibleDC
HDC CreateCompatibleDC(
  [in] HDC hdc
);
```

Now, we know the usage is fairly simple, we will provide the handle to the device context. The function usage will be as follows:

```cpp:driver.cpp
HDC hCompatibleDC = CreateCompatibleDC(hDC);
```

So far, our code will look something like this:

```cpp:screenshot.cpp
int sWidth  = GetSystemMetrics(SM_CXSCREEN);
int sHeight = GetSystemMetrics(SM_CYSCREEN);
HDC hDC = GetDC(NULL);
HDC hCompatibleDC = CreateCompatibleDC(hDC);
HBITMAP hBitmap = CreateCompatibleBitmap(hDC, sWidth, sHeight);
```

Now, the next thing we must do, is to select the bitmap into the compatible device context. We will be using the `SelectObject` function to do this. This function takes in two parameters which are the `hDC` which is a handle to the device context and `hObject` which is a handle to the object to be selected. The function returns a handle to the object being replaced if the function succeeds or `NULL` if the function fails.

```cpp:SelectObject
HGDIOBJ SelectObject(
  [in] HDC     hdc,
  [in] HGDIOBJ hgdiobj
);
```

Now, we know the usage is fairly simple, we will provide the handle to the compatible device context and the handle to the bitmap. The function usage will be as follows:

```cpp:driver.cpp
SelectObject(hCompatibleDC, hBitmap);

// Note, in some cases we will need to type cast in order to store it as another `bitmap`
HBITMAP hOldBitmap = (HBITMAP)SelectObject(hCompatibleDC, hBitmap);
```

The next thing which we must do, is to read the actual pixel values and store it in a `BITMAPINFOHEADER`. Let's take a look at what is a `BITMAPINFOHEADER`. According to [MSDN](https://docs.microsoft.com/en-us/windows/win32/api/wingdi/ns-wingdi-bitmapinfoheader)

> The BITMAPINFOHEADER structure contains information about the dimensions and color format of a device-independent bitmap (DIB).

There are only a few properties of the structure that we may utilize and modify, these

| Property | Description |
| --- | --- |
| biSize | The number of bytes required by the structure. |
| biWidth | The width of the bitmap, in pixels. |
| biHeight | The height of the bitmap, in pixels. |
| biPlanes | The number of planes for the target device. This value must be set to 1. |
| biBitCount | The number of bits-per-pixel. The biBitCount member of the BITMAPINFOHEADER structure determines the number of bits that define each pixel and the maximum number of colors in the bitmap. This member must be one of the following values. |
| biCompression | The type of compression for a compressed bottom-up bitmap (top-down DIBs cannot be compressed). This member can be one of the following values. |
| biSizeImage | The size, in bytes, of the image. This may be set to zero for BI_RGB bitmaps. |

The declaration for this struct will be as follows

```cpp:BITMAPINFOHEADER
BITMAPINFOHEADER bi;
bi.biSize = sizeof(BITMAPINFOHEADER);
bi.biWidth = screenWidth;
bi.biHeight = -screenHeight;
bi.biPlanes = 1;
bi.biBitCount = 24;
bi.biCompression = BI_RGB;
bi.biSizeImage = 0;
```

The reason why we set the `bi.biHeight` to `-screenHeight` is because the `BITMAPINFOHEADER` is a `bottom-up` bitmap. According to [MSDN](https://docs.microsoft.com/en-us/windows/win32/gdi/bottom-up-dibs)

> A bottom-up DIB is specified with a positive height, meaning that the first scan line is at the top of the bitmap and the last scan line is at the bottom. This is the most common orientation for Windows DIBs.

Also, we set the `biBitCount` to `24` because we want to store the pixel values in `RGB` format and for the same reason we set `biCompression` to `BI_RGB`.

The last thing we need to here is set the `BITMAPFILEHEADER` which is a structure that contains information about the type, size, and layout of a file that contains a DIB. Now, there are a lot of fields in this structure, but we will only be using the `bfType`, `bfSize`, `bfReserved1`, `bfReserved2`, and `bfOffBits`. Let's take a look at what these fields mean.

| Field | Description |
| --- | --- |
| bfType | Specifies the file type. It must be set to the signature word `BM` (0x4D42) to indicate bitmap. |
| bfSize | Specifies the size, in bytes, of the bitmap file. |
| bfReserved1 | Reserved; must be set to zero. |
| bfReserved2 | Reserved; must be set to zero. |
| bfOffBits | Specifies the offset, in bytes, from the beginning of the BITMAPFILEHEADER structure to the bitmap bits. |

The declaration for this struct will be as follows

```cpp:BITMAPFILEHEADER
BITMAPFILEHEADER bf;
bf.bfType = 0x4D42;
bf.bfSize = sizeof(BITMAPFILEHEADER) + sizeof(BITMAPINFOHEADER) + sWidth * sHeight * 3;
bf.bfReserved1 = 0;
bf.bfReserved2 = 0;
bf.bfOffBits = sizeof(BITMAPFILEHEADER) + sizeof(BITMAPINFOHEADER);
```

## Saving the image to a file or storing it in memory

### Saving the image to a file

Now, utilizing standard C++ `ofstream` to store the output to a file, but first, we will need to typecast the `BITMAPFILEHEADER` and the `BITMAPINFOHEADER` to `const char*`, that is because the `ofstream` will only accept `const char*` as an argument. Firstly, we'll be storing the `BITMAPFILEHEADER` to the file and then the BITMAPINFOHEADER and then the actual pixel values. The reason for doing this is

1. The `BITMAPFILEHEADER` contains the size of the entire file.
2. The `BITMAPINFOHEADER` contains the size of the actual image.
3. The actual pixel values.

```cpp:Storing in file
std::ofstream file("screenshot.bmp", std::ios::out | std::ios::binary);
if(file) { // Because we write code that handles errors too :)
    file.write(reinterpret_cast<const char*>(&bf), sizeof(BITMAPFILEHEADER));
    file.write(reinterpret_cast<const char*>(&bi), sizeof(BITMAPINFOHEADER));
    DWORD dwBmpSize = ((screenWidth * bi.biBitCount + 31) / 32) * 4 * screenHeight;
    BYTE* bmpData = new BYTE[dwBmpSize];
    GetDIBits(hMemoryDC, hBitmap, 0, screenHeight, bmpData, (BITMAPINFO*)&bi, DIB_RGB_COLORS);
    file.write(reinterpret_cast<const char*>(bmpData), dwBmpSize);
    delete[] bmpData;
    file.close();
}
```

### Storing it in memory

Similar to the previous one, we will be storing the data into a `vector` of `BYTE`, that is essentially a `typedef` of `char`. Now, we will be storing the `BITMAPFILEHEADER` to the vector and then the `BITMAPINFOHEADER` and then the actual pixel values as we did before but this time, into the `vector`. This time, we'll be storing the `BITMAPFILEHEADER` and the `BITMAPINFOHEADER` into their seperate variables of type `BYTE*`, that too because we need to insert them one after the another in the vector, and we'll be utilizing the `insert` method of the `vector` class instead of `push_back` for doing so. And before this, we will need the size of image, and the actual data. For that we will be utilizing [GetDIbits](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/nf-wingdi-getdibits) method. 

```cpp
DWORD dwBmpSize = ((screenWidth * bi.biBitCount + 31) / 32) * 4 * screenHeight;
BYTE* bmpData = new BYTE[dwBmpSize];
GetDIBits(hMemoryDC, hBitmap, 0, screenHeight, bmpData, (BITMAPINFO*)&bi, DIB_RGB_COLORS);
```

Therefore, the final code will become

```cpp
DWORD dwBmpSize = ((screenWidth * bi.biBitCount + 31) / 32) * 4 * screenHeight;
BYTE* bmpData = new BYTE[dwBmpSize];
GetDIBits(hMemoryDC, hBitmap, 0, screenHeight, bmpData, (BITMAPINFO*)&bi, DIB_RGB_COLORS);

BYTE* bfData = reinterpret_cast<BYTE*>(&bf);
BYTE* biData = reinterpret_cast<BYTE*>(&bi);
std::vector<BYTE> byteVector;
byteVector.reserve(sizeof(BITMAPFILEHEADER) + sizeof(BITMAPINFOHEADER) + dwBmpSize);
byteVector.insert(byteVector.end(), bfData, bfData + sizeof(BITMAPFILEHEADER));
byteVector.insert(byteVector.end(), biData, biData + sizeof(BITMAPINFOHEADER));
byteVector.insert(byteVector.end(), bmpData, bmpData + dwBmpSize);
delete[] bmpData;
```

### Deleting the objects and cleaning the code

After utilizing all the structure, bitmaps and device contexts, we need to delete them. We will be using the `DeleteObject` function to delete the bitmap and the `DeleteDC` function to delete the device context. The function usage will be as follows:

```cpp
DeleteObject(hBitmap);
DeleteDC(hCompatibleDC);
ReleaseDC(NULL, hDC);
```

## Putting it all together

Now, let's put it all together and see how it looks like.

### Storing in a file

```cpp:screenshot-file.cpp
#include <iostream>
#include <windows.h>
#include <fstream>

int main() {
    
    int sWidth = GetSystemMetrics(SM_CXSCREEN);
    int sHeight  = GetSystemMetrics(SM_CYSCREEN);

    HDC hDC = GetDC(NULL);
    HDC hMemoryDC = CreateCompatibleDC(hDC);
    HBITMAP hBitmap = CreateCompatibleBitmap(hDC, sWidth, sHeight);
    HBITMAP hOldBitmap = (HBITMAP)SelectObject(hMemoryDC, hBitmap);

    BitBlt(hMemoryDC, 0, 0, sWidth, sHeight, hDC, 0, 0, SRCCOPY);

    BITMAPINFOHEADER bi;
    bi.biSize = sizeof(BITMAPINFOHEADER);
    bi.biWidth = sWidth;
    bi.biHeight = -sHeight;
    bi.biPlanes = 1;
    bi.biBitCount = 24;
    bi.biCompression = BI_RGB;
    bi.biSizeImage = 0;

    BITMAPFILEHEADER bf;
    bf.bfType = 0x4D42;
    bf.bfSize = sizeof(BITMAPFILEHEADER) + sizeof(BITMAPINFOHEADER) + sWidth * sHeight * 3;
    bf.bfReserved1 = 0;
    bf.bfReserved2 = 0;
    bf.bfOffBits = sizeof(BITMAPFILEHEADER) + sizeof(BITMAPINFOHEADER);

    std::ofstream file("screenshot.bmp", std::ios::out | std::ios::binary);
    if (file) {
        file.write(reinterpret_cast<const char*>(&bf), sizeof(BITMAPFILEHEADER));
        file.write(reinterpret_cast<const char*>(&bi), sizeof(BITMAPINFOHEADER));
        DWORD dwBmpSize = ((sWidth * bi.biBitCount + 31) / 32) * 4 * sHeight;
        BYTE* bmpData = new BYTE[dwBmpSize];
        GetDIBits(hMemoryDC, hBitmap, 0, sHeight, bmpData, (BITMAPINFO*)&bi, DIB_RGB_COLORS);
        file.write(reinterpret_cast<const char*>(bmpData), dwBmpSize);
        delete[] bmpData;
        file.close();
    }

    SelectObject(hMemoryDC, hOldBitmap);
    DeleteObject(hBitmap);
    DeleteDC(hMemoryDC);
    ReleaseDC(NULL, hDC);

    return 0;
}

```

### Storing in memory

```cpp:screenshot-memory.cpp
#include <iostream>
#include <windows.h>
#include <fstream>
#include <vector>

int main() {
    
    int sWidth = GetSystemMetrics(SM_CXSCREEN);
    int sHeight  = GetSystemMetrics(SM_CYSCREEN);

    HDC hDC = GetDC(NULL);
    HDC hMemoryDC = CreateCompatibleDC(hDC);
    HBITMAP hBitmap = CreateCompatibleBitmap(hDC, sWidth, sHeight);
    HBITMAP hOldBitmap = (HBITMAP)SelectObject(hMemoryDC, hBitmap);

    BitBlt(hMemoryDC, 0, 0, sWidth, sHeight, hDC, 0, 0, SRCCOPY);

    BITMAPINFOHEADER bi;
    bi.biSize = sizeof(BITMAPINFOHEADER);
    bi.biWidth = sWidth;
    bi.biHeight = -sHeight;
    bi.biPlanes = 1;
    bi.biBitCount = 24;
    bi.biCompression = BI_RGB;
    bi.biSizeImage = 0;

    BITMAPFILEHEADER bf;
    bf.bfType = 0x4D42;
    bf.bfSize = sizeof(BITMAPFILEHEADER) + sizeof(BITMAPINFOHEADER) + sWidth * sHeight * 3;
    bf.bfReserved1 = 0;
    bf.bfReserved2 = 0;
    bf.bfOffBits = sizeof(BITMAPFILEHEADER) + sizeof(BITMAPINFOHEADER);

    DWORD dwBmpSize = ((sWidth * bi.biBitCount + 31) / 32) * 4 * sHeight;
    BYTE* bmpData = new BYTE[dwBmpSize];
    GetDIBits(hMemoryDC, hBitmap, 0, sHeight, bmpData, (BITMAPINFO*)&bi, DIB_RGB_COLORS);

    BYTE* bfData = reinterpret_cast<BYTE*>(&bf);
    BYTE* biData = reinterpret_cast<BYTE*>(&bi);
    std::vector<BYTE> imageVec;
    imageVec.reserve(sizeof(BITMAPFILEHEADER) + sizeof(BITMAPINFOHEADER) + dwBmpSize);
    imageVec.insert(imageVec.end(), bfData, bfData + sizeof(BITMAPFILEHEADER));
    imageVec.insert(imageVec.end(), biData, biData + sizeof(BITMAPINFOHEADER));
    imageVec.insert(imageVec.end(), bmpData, bmpData + dwBmpSize);

    delete[] bmpData;

    SelectObject(hMemoryDC, hOldBitmap);
    DeleteObject(hBitmap);
    DeleteDC(hMemoryDC);
    ReleaseDC(NULL, hDC);

    // use the imageVec to further do whatever you want, SEND it over sockets, store, etc...

    return 0;
}

```

## Moving Further

For those that know me, and those who don't, I really like utilizing Object Oriented concepts and organzing my code into modules that I can simply plug-n-play in my future projects without setting anything up. This, can be used in different projects else where. So, after converting this a seperate namespace, and dividing both of these into a single header file and different methods to do different things, I came up with this:

```cpp:screenshots.hpp
/*
    Author: @TheFlash2k
    Github: https://github.com/theflash2k/
*/

#ifndef SCREENSHOT_HPP
#define SCREENSHOT_HPP

#include <windows.h>
#include <iostream>
#include <fstream>
#include <vector>

namespace Screenshot {

    // I like naming my private namespaces with __internal to hide the functions that the user might not need.
    // More like abstraction but, not really. If only private namespaces were a thing.
    namespace __internal {
        typedef struct {
            int sWidth;
            int sHeight;
            HDC hDC;
            HDC hMemoryDC;
            HBITMAP hBitmap;
            HBITMAP hOldBitmap;
            BITMAPINFOHEADER bi;
            BITMAPFILEHEADER bf;
            DWORD dwBmpSize;
            BYTE* bmpData;
        }SCREENSHOT, * PSCREENSHOT;

        // This method initializes the necessary components for taking screenshots
        void init_screenshot(PSCREENSHOT ss) {
            ss->sWidth = GetSystemMetrics(SM_CXSCREEN);
            ss->sHeight = GetSystemMetrics(SM_CYSCREEN);
            ss->hDC = GetDC(NULL);
            ss->hMemoryDC = CreateCompatibleDC(ss->hDC);
            ss->hBitmap = CreateCompatibleBitmap(ss->hDC, ss->sWidth, ss->sHeight);
            ss->hOldBitmap = (HBITMAP)SelectObject(ss->hMemoryDC, ss->hBitmap);
            BitBlt(ss->hMemoryDC, 0, 0, ss->sWidth, ss->sHeight, ss->hDC, 0, 0, SRCCOPY);
            ss->bi.biSize = sizeof(BITMAPINFOHEADER);
            ss->bi.biWidth = ss->sWidth;
            ss->bi.biHeight = -ss->sHeight;
            ss->bi.biPlanes = 1;
            ss->bi.biBitCount = 24;
            ss->bi.biCompression = BI_RGB;
            ss->bi.biSizeImage = 0;
            ss->dwBmpSize = ((ss->sWidth * ss->bi.biBitCount + 31) / 32) * 4 * ss->sHeight;
            ss->bmpData = new BYTE[ss->dwBmpSize];
            GetDIBits(ss->hMemoryDC, ss->hBitmap, 0, ss->sHeight, ss->bmpData, (BITMAPINFO*)&ss->bi, DIB_RGB_COLORS);
            ss->bf.bfType = 0x4D42;
            ss->bf.bfSize = sizeof(BITMAPFILEHEADER) + sizeof(BITMAPINFOHEADER) + ss->dwBmpSize;
            ss->bf.bfReserved1 = 0;
            ss->bf.bfReserved2 = 0;
            ss->bf.bfOffBits = sizeof(BITMAPFILEHEADER) + sizeof(BITMAPINFOHEADER);
        }

        void delete_screenshot(PSCREENSHOT ss) {
            delete[] ss->bmpData;
            SelectObject(ss->hMemoryDC, ss->hOldBitmap);
            DeleteObject(ss->hBitmap);
            DeleteDC(ss->hMemoryDC);
            ReleaseDC(NULL, ss->hDC);
        }

    }

    std::vector<BYTE> GetCurrentState() {
        using namespace __internal;
        SCREENSHOT ss;
        init_screenshot(&ss);
        BYTE* bfData = reinterpret_cast<BYTE*>(&ss.bf);
        BYTE* biData = reinterpret_cast<BYTE*>(&ss.bi);
        std::vector<BYTE> byteVector;
        byteVector.reserve(sizeof(BITMAPFILEHEADER) + sizeof(BITMAPINFOHEADER) + ss.dwBmpSize);
        byteVector.insert(byteVector.end(), bfData, bfData + sizeof(BITMAPFILEHEADER));
        byteVector.insert(byteVector.end(), biData, biData + sizeof(BITMAPINFOHEADER));
        byteVector.insert(byteVector.end(), ss.bmpData, ss.bmpData + ss.dwBmpSize);
        delete_screenshot(&ss);
        return byteVector;
    }

    // Another method to store the screenshot vector to a file.
    bool ToFile(std::vector<BYTE> ssVec, std::string fileName = "output.bmp") {
		using namespace __internal;
        if (fileName.substr(fileName.size() - 4, 4) != ".bmp") {
            fileName += ".bmp";
        }
        std::ofstream file(fileName.c_str(), std::ios::out | std::ios::binary);
        if (file) {
			file.write(reinterpret_cast<const char*>(&ssVec[0]), ssVec.size());
			file.close();
            return true;
		}
        return false;
    }

    // Capturing a screenshot, and storing the output to a file.
    bool ToFile(std::string fileName="output.bmp") {
        using namespace __internal;
        if (fileName.substr(fileName.size() - 4, 4) != ".bmp") {
			fileName += ".bmp";
		}
        SCREENSHOT ss;
        init_screenshot(&ss);
        std::ofstream file(fileName.c_str(), std::ios::out | std::ios::binary);
        if (file) {
            file.write(reinterpret_cast<const char*>(&ss.bf), sizeof(BITMAPFILEHEADER));
            file.write(reinterpret_cast<const char*>(&ss.bi), sizeof(BITMAPINFOHEADER));
            file.write(reinterpret_cast<const char*>(ss.bmpData), ss.dwBmpSize);
            file.close();
            return true;
        }
        delete_screenshot(&ss);
        return false;
    }
}
#endif // !SCREENSHOT_HPP
```

NOTE: This [source code](https://github.com/TheFlash2k/winapi-projects/blob/main/screenshots.hpp) can be found on my [GitHub](https://github.com/TheFlash2k/).

The main driver file will look something like this:

```cpp:driver.cpp
#include "screenshots.hpp"

int main(int argc, char* argv[]) {
     // Storing to a file
    Screenshot::ToFile("test.bmp");

    // Storing to a vector
    std::vector<BYTE> ssVec = Screenshot::GetCurrentState();

    // Storing the vector to a file
    Screenshot::ToFile(ssVec, "test_vec.bmp");

    return 0;
}
```

## Conclusion

In this post, we took a look at how we can take screenshots using Windows API in C++. The output will be in BMP format and can be stored in either a file or in memory. We also took a look at how we can organize our code into modules and make it more readable and reusable. I hope you enjoyed this post and learned something new. If you have any questions, feel free to reach out to me on [Twitter](https://twitter.com/theflash2k) or [LinkedIn](https://www.linkedin.com/in/theflash2k/).
