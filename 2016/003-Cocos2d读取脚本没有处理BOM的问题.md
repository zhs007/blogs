# Cocos2d读取脚本没有处理BOM的问题

Why
---
因为我们项目组是跨平台开发，一部分同学在windows下开发，另外一批在mac下开发，windows下很多文本编辑器默认字符集都是本地字符集（gbk），gbk和utf8之间的自动识别其实是存在一些问题的，所以才有了BOM这种东东。

我们要求所有的本地编码都是带BOM的utf8，这样跨平台编辑会方便一些，毕竟现在用的这一批编辑器都能正确的识别BOM。


解决方案
---
为了一劳永逸，我们干脆来修改``FileUtils``吧，因为cocos2d所有的文件读取其实都是走的这个（不同系统下这个是不一样的哦），如果要处理BOM，我们可以在这个文件里面，读取文件以后自己判断一下BOM即可。

考虑到这是一个底层接口，不要去动返回内存空间大小之类的，所以我们简单一点，把BOM的3个字节改成空格就好了。

```
    if (buffer[0] == 0xef && buffer[1] == 0xbb && buffer[2] == 0xbf) {
        buffer[0] = ' ';
        buffer[1] = ' ';
        buffer[2] = ' ';
    }
```

这里简单的提一下，一般来说，如果要修改引擎代码（不是我们自己写的引擎），最好修改的地方用特殊注释标识出来，这样方便后面更新引擎版本时处理我们自己的修正补丁。

我们一般都是类似下面的编码方式：

```
    //<-- zhs007 @ 160111
    // 如果是字符串，处理掉BOM
    if (forString) {
        if (buffer[0] == 0xef && buffer[1] == 0xbb && buffer[2] == 0xbf) {
            buffer[0] = ' ';
            buffer[1] = ' ';
            buffer[2] = ' ';
        }
    }
    //<-- end
```

我改了``getFileDataFromZip``、``getFileData``、``getData``3个接口，其实，理论上我们只需要修改字符串读取的文件，二进制文件就不需要处理了。

``getData``接口里面有个参数是``bool forString``，判断这个就好了

```
static Data getData(const std::string& filename, bool forString)
{	
    if (filename.empty())
    {
        return Data::Null;
    }
    
    Data ret;
    unsigned char* buffer = nullptr;
    size_t size = 0;
    size_t readsize;
    const char* mode = nullptr;
    
    if (forString)
        mode = "rt";
    else
        mode = "rb";
    
    do
    {
        // Read the file from hardware
        std::string fullPath = FileUtils::getInstance()->fullPathForFilename(filename,false);
        FILE *fp = fopen(fullPath.c_str(), mode);
        CC_BREAK_IF(!fp);
        fseek(fp,0,SEEK_END);
        size = ftell(fp);
        fseek(fp,0,SEEK_SET);
        
        if (forString)
        {
            buffer = (unsigned char*)malloc(sizeof(unsigned char) * (size + 1));
            buffer[size] = '\0';
        }
        else
        {
            buffer = (unsigned char*)malloc(sizeof(unsigned char) * size);
        }
        
        readsize = fread(buffer, sizeof(unsigned char), size, fp);
        fclose(fp);
        
        if (forString && readsize < size)
        {
            buffer[readsize] = '\0';
        }
    } while (0);
    
    //<-- zhs007 @ 160111
    // 如果是字符串，处理掉BOM
    // 注意：实测发现内部大量的脚本读取是二进制方式，因此屏蔽掉判断
    //if (forString) {
        if (buffer[0] == 0xef && buffer[1] == 0xbb && buffer[2] == 0xbf) {
            buffer[0] = ' ';
            buffer[1] = ' ';
            buffer[2] = ' ';
        }
    //}
    //<-- end
    
    if (nullptr == buffer || 0 == readsize)
    {
        std::string msg = "Get data from file(";
        msg.append(filename).append(") failed!");
        CCLOG("%s", msg.c_str());
    }
    else
    {
        ret.fastSet(buffer, readsize);
    }
    
    return ret;
}
```

``getFileData``接口只能判断``const char* mode``了，C里面，如果带``b``才是二进制读取。

```
unsigned char* FileUtils::getFileData(const std::string& filename, const char* mode, ssize_t *size)
{
    unsigned char * buffer = nullptr;
    CCASSERT(!filename.empty() && size != nullptr && mode != nullptr, "Invalid parameters.");
    *size = 0;
    do
    {
        // read the file from hardware
        const std::string fullPath = fullPathForFilename(filename);
        FILE *fp = fopen(fullPath.c_str(), mode);
        CC_BREAK_IF(!fp);
        
        fseek(fp,0,SEEK_END);
        *size = ftell(fp);
        fseek(fp,0,SEEK_SET);
        buffer = (unsigned char*)malloc(*size);
        *size = fread(buffer,sizeof(unsigned char), *size,fp);
        fclose(fp);

        //<-- zhs007 @ 160111
        // 如果不是二进制读取方式，需要处理BOM
        // 注意：实测发现内部大量的脚本读取是二进制方式，因此屏蔽掉判断
        //if (strstr(mode, "b") != NULL) {
            if (buffer[0] == 0xef && buffer[1] == 0xbb && buffer[2] == 0xbf) {
                buffer[0] = ' ';
                buffer[1] = ' ';
                buffer[2] = ' ';
            }
        //}
        //<-- end
    } while (0);
    
    if (!buffer)
    {
        std::string msg = "Get data from file(";
        msg.append(filename).append(") failed!");
        
        CCLOG("%s", msg.c_str());
    }
    return buffer;
}
```

``getFileDataFromZip``这个接口没办法判断是否是二进制读取，而且我们没用这个接口，其实理论上即便是二进制文件，一般来说前3个字符正好是BOM的概率很小很小（二进制文件前面一般都是文件头），所以干脆简单点处理掉算了。

```
unsigned char* FileUtils::getFileDataFromZip(const std::string& zipFilePath, const std::string& filename, ssize_t *size)
{
    unsigned char * buffer = nullptr;
    unzFile file = nullptr;
    *size = 0;

    do 
    {
        CC_BREAK_IF(zipFilePath.empty());

        file = unzOpen(zipFilePath.c_str());
        CC_BREAK_IF(!file);

        // FIXME: Other platforms should use upstream minizip like mingw-w64  
        #ifdef MINIZIP_FROM_SYSTEM
        int ret = unzLocateFile(file, filename.c_str(), NULL);
        #else
        int ret = unzLocateFile(file, filename.c_str(), 1);
        #endif
        CC_BREAK_IF(UNZ_OK != ret);

        char filePathA[260];
        unz_file_info fileInfo;
        ret = unzGetCurrentFileInfo(file, &fileInfo, filePathA, sizeof(filePathA), nullptr, 0, nullptr, 0);
        CC_BREAK_IF(UNZ_OK != ret);

        ret = unzOpenCurrentFile(file);
        CC_BREAK_IF(UNZ_OK != ret);

        buffer = (unsigned char*)malloc(fileInfo.uncompressed_size);
        int CC_UNUSED readedSize = unzReadCurrentFile(file, buffer, static_cast<unsigned>(fileInfo.uncompressed_size));
        CCASSERT(readedSize == 0 || readedSize == (int)fileInfo.uncompressed_size, "the file size is wrong");

        *size = fileInfo.uncompressed_size;
        unzCloseCurrentFile(file);
    } while (0);

    if (file)
    {
        unzClose(file);
    }
    
    //<-- zhs007 @ 160111
    // 注意：分辨不出是否二进制读取，优先处理掉BOM
    if (buffer[0] == 0xef && buffer[1] == 0xbb && buffer[2] == 0xbf) {
        buffer[0] = ' ';
        buffer[1] = ' ';
        buffer[2] = ' ';
    }
    //<-- end

    return buffer;
}
```