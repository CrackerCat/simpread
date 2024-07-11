> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282460.htm#msg_header_h2_1)

> [原创] 手搓函数抽取加固 (上)

本次研究基于安卓 12

基础篇 --dex 结构解析
--------------

### 工具

对于 dex 文件结构的的解析，我认为 c 或者 c++ 语言是比较好的

推荐工具 c 语言的编辑工具 ---- 我是用的 qt 简单方便

系统: windows 11

验证工具： 010 editor

### dex 文件大致结构

文件头:**header**

索引区:**string_ids,type_ids,proto_ids,field_ids,method_ids**

数据区:**class_defs,data,linke_data**

### 0x1 header 解析

了解结构体

```
// Raw header_item.
struct Header {
      uint8_t magic_[8] = {};
      uint32_t checksum_ = 0;  // See also location_checksum_
      uint8_t signature_[20] = {};
      uint32_t file_size_ = 0;  // size of entire file
      uint32_t header_size_ = 0;  // offset to start of next section
      uint32_t endian_tag_ = 0;
      uint32_t link_size_ = 0;  // unused
      uint32_t link_off_ = 0;  // unused
  uint32_t map_off_ = 0;  // map list offset from data_off_
  uint32_t string_ids_size_ = 0;  // number of StringIds
  uint32_t string_ids_off_ = 0;  // file offset of StringIds array
  uint32_t type_ids_size_ = 0;  // number of TypeIds, we don't support more than 65535
  uint32_t type_ids_off_ = 0;  // file offset of TypeIds array
  uint32_t proto_ids_size_ = 0;  // number of ProtoIds, we don't support more than 65535
  uint32_t proto_ids_off_ = 0;  // file offset of ProtoIds array
// file offset of data section
      // Decode the dex magic version
     // uint32_t GetVersion() const;
};

```

头部结构体 AI 解读：

1.  **`magic_[8]`**：这是一个 8 字节的数组，用于存储文件的魔数（magic number）。魔数通常用于标识文件的类型或格式，确保读取文件的程序能够识别出这是一个它所能处理的文件。
    
2.  **`checksum_`**：这是一个 32 位的校验和，用于验证文件内容的完整性。在文件传输或存储过程中，如果文件内容被修改，这个校验和将不再匹配，从而可以检测到文件已被篡改。
    
3.  **`signature_[20]`**：这是一个 20 字节的签名数组，具体用途可能依赖于文件的实际使用场景。在某些情况下，它可能用于身份验证或完整性检查。
    
4.  **`file_size_`**：整个文件的大小（以字节为单位）。这个信息对于了解文件的整体大小很有帮助，尤其是在需要预先分配内存或进行文件操作时。
    
5.  **`header_size_`**：头部的大小（以字节为单位），或者是头部结束后，下一个部分的起始偏移量。这个信息对于解析文件的其他部分非常关键。
    
6.  **`endian_tag_`**：这个字段可能用于指示文件中数据的字节序（大端或小端）。这对于跨平台的数据交换和解析非常重要。
    
7.  **`link_size_` 和 `link_off_`**：这两个字段当前标记为未使用（unused），但在某些文件格式中，它们可能用于链接到文件的其他部分或提供额外的元数据。
    
8.  **`map_off_`**：映射列表的偏移量，从`data_off_`开始计算。映射列表可能包含了文件中各个部分的详细描述或索引。
    
9.  **字符串 ID、类型 ID、原型 ID、字段 ID、方法 ID 和类定义的相关字段**（如`string_ids_size_`, `string_ids_off_`, `type_ids_size_`等）：这些字段提供了关于文件中不同类型 ID、字符串、类定义等信息的数量和偏移量。这对于解析文件内容和构建程序的内部表示非常重要。
    
10.  **`data_size_` 和 `data_off_`**：分别表示数据部分的大小和偏移量。数据部分可能包含了文件的主要执行内容或数据。
    

解析代码：

```
int main()
{
    FILE *fp = fopen("C:\\Users\\Administrator\\Desktop\\dex\\classes4.dex","rb");

    if(fp == NULL){
        log("失败！ error! fopen");

    }{
        log("fopen success!");
    }
    Header *header =(Header *) malloc(sizeof(Header));
    fread((void *)header,sizeof(char),sizeof(Header),fp);
    showHeader(header);

    return 0;
}
void showHeader(Header *header){
    printf("******************print begin*****************\n");
    printf("magic: \t\t %s \n",header->magic_);
    printf("checksum: \t\t %lx\n",header->checksum_);
    printf("file size: \t\t %x\n",header->file_size_);
    printf("head size: \t\t %x\n",header->header_size_);
    printf("endian tag: \t\t %x\n",header->endian_tag_);
    printf("link size: \t\t %x\n",header->link_size_);
    printf("link off: \t\t %x\n",header->link_off_);
    printf("map off: \t\t %x\n",header->link_off_);
    printf("string ids size: \t %x\n",header->string_ids_size_);
    printf("string ids off: \t %x\n",header->string_ids_off_);
    printf("type ids size : \t %x\n",header->type_ids_size_);
    printf("type ids off:\t \t %x\n",header->type_ids_off_);
    printf("proto ids size: \t %x\n",header->proto_ids_size_);
    printf("proto ids off: \t\t %x\n",header->proto_ids_off_);
    printf("field ida size: \t %x\n",header->field_ids_size_);
    printf("field ida off:\t \t %x\n",header->field_ids_off_);
    printf("method ids size: \t %x\n",header->method_ids_size_);
    printf("method ids off: \t %x\n",header->method_ids_off_);
    printf("class defs size: \t %x\n",header->class_defs_size_);
    printf("class defs off: \t %x\n",header->class_defs_off_);
    printf("data size:\t \t %x\n",header->data_size_);
    printf("data off: \t\t %x\n",header->data_off_);
    printf("*****************header over print************\n");
}

```

### 0x2 **string_ids 解析**

接下来解析字符串常量池

从头部获取 string_ids_off 与 size

即可解析  先看这个结构体吧：

```
struct DexStringId {
    uint32_t stringDataOff;   /* 字符串数据偏移 */
};

```

这个结构体告诉你字符串的地址

```
DexStringId * string_ids(Header *header);
FILE *fp;
DexStringId  * strings;
char * buffer;
Header *header;
void  init_string(){
    strings = string_ids(header);
    buffer = (char *)malloc(header->file_size_);
    fseek(fp,0,0);
    fread((void*)buffer,sizeof(char),header->file_size_,fp);
}

char * getString(int index){
    return &buffer[strings[index].stringDataOff+1];
}

```

### 0x3 type_ids 解析

老规矩先看结构体：

```
struct DexTypeId {
    uint32_t  descriptorIdx;      /* 指向DexStringId列表的索引 */
};

```

结构体指向的数据 是字符串常量池的索引

```
char * gettypestring(int index){
    return getString(types[index].descriptorIdx);
}

void init_type(){
    types =(DexTypeId *) malloc(header->type_ids_size_*sizeof(DexTypeId));
    fseek(fp,header->type_ids_off_,0);
    fread((void *)types,sizeof(DexTypeId),header->type_ids_size_,fp);
}

```

打印全部 type 的代码：

```
//初始化

    init_type();
    //打印所有types
    for(int i = 0;i<header->type_ids_size_;i++){
        printf("%s\n",gettypestring(i));
    }

```

### 0x4 解析 proto ids

观摩结构体

```
struct DexProtoId {
    uint32_t shortyIdx;   /* 指向DexStringId列表的索引 */
    uint32_t returnTypeIdx;   /* 指向DexTypeId列表的索引 */
    uint32_t parametersOff;   /* 指向DexTypeList的偏移 */
};

struct DexTypeList {
    uint32_t size;             /* 接下来DexTypeItem的个数 */
    DexTypeItem list[1]; /* DexTypeItem结构 */
};

struct DexTypeItem {
    uint16_t typeIdx;    /* 指向DexTypeId列表的索引 */
};

```

解析 protos

```
void show_protos(){
    for(int i=0;i<header->proto_ids_size_;i++){
        printf("%s \n",getString(protos[i].shortyIdx));
        printf("%s \n",gettypestring(protos[i].returnTypeIdx));

        if(protos[i].parametersOff != 0){
            DexTypeList *list =(DexTypeList *) (buffer+protos[i].parametersOff) ;
            for(int a=0;a<list->size;a++){
                printf("%s \n",gettypestring(list->list[a].typeIdx));
            }
        }
    }
}

void init_protos(){
    //初始化DexProtoId
    protos=(DexProtoId *) malloc(header->proto_ids_size_*sizeof(DexProtoId));
    fseek(fp,header->proto_ids_off_,0);
    fread((void *)protos,sizeof(DexProtoId),header->proto_ids_size_,fp);

}

```

### 0x5 解析 fields

解析成员变量

上结构体

```
struct DexFieldId {
    uint16_t classIdx;   /* 类的类型，指向DexTypeId列表的索引 */
    uint16_t typeIdx;    /* 字段类型，指向DexTypeId列表的索引 */
    uint32_t nameIdx;    /* 字段名，指向DexStringId列表的索引 */
};

```

解析打印

```
void show_fields(){
    for(int i=0;i<header->field_ids_size_;i++){
        printf("所在类：%s，成员类型 %s,变量名 %s \n",gettypestring(fields[i].classIdx),gettypestring(fields[i].typeIdx),getString(fields[i].nameIdx));
    }
}

void init_fields(){
    fields = (DexFieldId *)malloc (header->field_ids_size_*sizeof(DexFieldId));
    fseek(fp,header->field_ids_off_,0);
    fread((void*)fields,sizeof(DexFieldId),header->field_ids_size_,fp);
}

```

### 0x6 解析 method_ids

结构体：

```
struct DexMethodId {
    uint16_t classIdx;  /* 类的类型，指向DexTypeId列表的索引 */
    uint16_t protoIdx;  /* 声明类型，指向DexProtoId列表的索引 */
    uint32_t nameIdx;   /* 方法名，指向DexStringId列表的索引 */
};

```

解析代码：

```
void show_methodids(){
    for(int i=0;i<header->method_ids_size_;i++){
        printf("%s  \n",gettypestring(methodids[i].classIdx));
        getprotosstring(methodids[i].protoIdx);
        printf("%s \n",getString(methodids[i].nameIdx));
    }
}
void init_methodids(){
    methodids=(DexMethodId *)malloc(header->method_ids_size_*sizeof(DexMethodId));
    fseek(fp,header->method_ids_off_,0);
    fread((void *)methodids,sizeof(DexMethodId),header->method_ids_size_,fp);
}
void  getprotosstring(int i){
    printf("%s ",getString(protos[i].shortyIdx));
    printf("%s ",gettypestring(protos[i].returnTypeIdx));

    if(protos[i].parametersOff != 0){
        DexTypeList *list =(DexTypeList *) (buffer+protos[i].parametersOff) ;
        for(int a=0;a<list->size;a++){
            printf("%s ",gettypestring(list->list[a].typeIdx));
        }

    }
}

```

### 0x7 解析 class

```
struct class_def_item
{
    uint32_t class_idx;         //描述具体的 class 类型，值是 type_ids 的一个 index 。值必须是一个 class 类型，不能是数组类型或者基本类型。   
    uint32_t access_flags;        //描述 class 的访问类型，诸如 public , final , static 等。在 dex-format.html 里 “access_flags Definitions” 有具体的描述 
    uint32_t superclass_idx;    //描述 supperclass 的类型，值的形式跟 class_idx 一样 
    uint32_t interface_off;     //值为偏移地址，指向 class 的 interfaces，被指向的数据结构为 type_list 。class 若没有 interfaces 值为 0
    uint32_t source_file_idx;    //表示源代码文件的信息，值是 string_ids 的一个 index。若此项信息缺失，此项值赋值为 NO_INDEX=0xffff ffff
    uint32_t annotations_off;    //值是一个偏移地址，指向的内容是该 class 的注释，位置在 data 区，格式为 annotations_direcotry_item。若没有此项内容，值为 0 
    uint32_t class_data_off;    //值是一个偏移地址，指向的内容是该 class 的使用到的数据，位置在 data 区，格式为 class_data_item。若没有此项内容值为 0。该结构里有很多内容，详细描述该 class 的 field、method, method 里的执行代码等信息，后面会介绍 class_data_item
    uint32_t static_value_off;    //值是一个偏移地址 ，指向 data 区里的一个列表 (list)，格式为 encoded_array_item。若没有此项内容值为 0
};

```

1.  **`uint class_idx;`**

*   这是一个索引值，指向 DEX 文件中的`type_ids`列表。`type_ids`列表包含了 DEX 文件中所有引用的类型（类、接口等）的引用。`class_idx`的值必须是一个指向类类型的索引，不能是数组类型或基本类型的索引。这个字段用于指定当前`class_def_item`所描述的类的类型。

1.  **`uint access_flags;`**

*   这个字段包含了类的访问标志，如`public`、`final`、`static`等。这些标志决定了类的可见性和行为。访问标志的具体含义在 DEX 文件格式文档中 “access_flags Definitions” 部分有详细描述。

1.  **`uint superclass_idx;`**

*   类似于`class_idx`，这也是一个索引值，指向`type_ids`列表。它指定了当前类的父类类型。如果当前类没有父类（比如`java.lang.Object`），则此字段的值为特定的索引值（通常是`type_ids`列表的第一个元素，即`java.lang.Object`）。

1.  **`uint interface_off;`**

*   这是一个偏移量，指向 DEX 文件中的`type_list`结构，该结构包含了当前类实现的所有接口的引用。如果当前类没有实现任何接口，则此字段的值为 0。

1.  **`uint source_file_idx;`**

*   这是一个索引值，指向 DEX 文件中的`string_ids`列表，表示定义当前类的源代码文件的名称。如果源代码文件信息缺失，则此字段的值被设置为`NO_INDEX`（通常是`0xffffffff`）。

1.  **`uint annotations_off;`**

*   这是一个偏移量，指向 DEX 文件中`annotations_directory_item`结构的位置，该结构包含了当前类的注解信息。如果当前类没有注解，则此字段的值为 0。

1.  **`uint class_data_off;`**

*   这是一个偏移量，指向 DEX 文件中`class_data_item`结构的位置，该结构包含了当前类的详细信息，如字段（field）、方法（method）、方法的执行代码（code）等。这是理解类内部结构和行为的关键部分。如果当前类不包含这些方法级别的信息（例如，它是一个标记为`abstract`的接口），则此字段的值为 0。

1.  **`uint static_values_off;`**

*   这是一个偏移量，指向 DEX 文件中`encoded_array_item`结构的位置，该结构包含了当前类的静态字段的初始值。如果当前类没有静态字段或没有为静态字段提供初始值，则此字段的值为 0。

在我们重点关注**`class_data_off`**指向的结构吧

先看看 **`class_data_off`**指向的结构体吧

```
struct class_data_item
{
    uleb128 static_fields_size; //静态成员变量的个数
    uleb128 instance_fields_size; //实例成员变量个数
    uleb128 direct_methods_size; //直接函数个数
    uleb128 virtual_methods_size; // 虚函数个数
    encoded_field  static_fields[static_fields_size];
    encoded_field  instance_fields[instance_fields_size];
    encoded_method direct_methods[direct_methods_size];
    encoded_method virtual_methods[virtual_methods_size];
}

```

这里使用了 uleb128 的方式存储 那我们在使用结构体直接覆盖得到数据的方式就不在适用了，那这个结构体不导入进项目！

那自己设计一个函数  要求获取读取的数据值，还有在内存中的大小 就可以比较方便的读取了

```
int read_uleb128(uint8_t * addr,int *size){
    int result= 0;
    *size =0;
    result = addr[0] & 0x7f;
    *size++;
    if(addr[0] & 0x80 ){
        result = result+(addr[1] & 0x7f)<<7;
        *size++;
        if((addr[1]& 0x80)){
             result = result+(addr[2] & 0x7f)<<14;
            *size++;
            if((addr[2]& 0x80)){
                result = result+(addr[3] & 0x7f)<<21;
                *size++;
            } 
        }        
    }
    return result;
}

```

有了这个工具就可以更好的解析了！

我们通过 uleb128 读取获取了

static_fields_size; // 静态成员变量的个数

instance_fields_size; // 实例成员变量个数

direct_methods_size; // 直接函数个数

virtual_methods_size; // 虚函数个数

```
 for(int i=0;i<header->class_defs_size_;i++){
        printf("data_off %d \n",classitems[i].class_data_off);
        void *addr = (void *)(&buffer[classitems[i].class_data_off]);
        //printf(" %2x %2x %2x %2x \n",(int8_t)buffer[classitems[i].class_data_off],(int8_t)buffer[classitems[i].class_data_off+1],(int8_t)buffer[classitems[i].class_data_off+2],(int8_t)buffer[classitems[i].class_data_off+3]);
        int size=0;
        int static_fields_size =  read_uleb128((uint8_t *)addr,&size);//静态成员变量的个数
        addr+=size;
        int instance_fields_size = read_uleb128((uint8_t *)addr,&size);//实例成员变量个数
        addr+=size;
        int direct_methods_size = read_uleb128((uint8_t *)addr,&size); //直接函数个数
        addr+=size;
        int virtual_methods_size = read_uleb128((uint8_t *)addr,&size);// 虚函数个数
        addr+=size;

        printf("静态成员变量:%d ,实例成员变量个数%d,直接函数个数%d.虚函数个数%d\n ",static_fields_size,instance_fields_size,direct_methods_size,virtual_methods_size);
 }
void init_class_def_item(){
    classitems = (class_def_item *)malloc(header->class_defs_size_*sizeof(class_def_item));
    fseek(fp,header->class_defs_off_,0);
    fread((void *)classitems,sizeof(class_def_item),header->class_defs_size_,fp);
}

```

然后按顺序解析

field 与 method

这里的结构体也是 uleb128

```
struct DexField{
    uleb128 fieldIdx;//指向DexFieldId索引
    uleb128 axxessFlags;//字段访问标志
}
struct DexMethod{
    uleb128 methodIdx;//指向DexMethodId索引
    uleb128 accessFlags;//方法访问标志
    uleb128 codeOff;//指向DexCode结构偏移
}

```

继续在原来的方法上解析

放解析代码：

```
void jx_class_data(){
    for(int i=0;i<header->class_defs_size_;i++){
        printf("data_off %d \n",classitems[i].class_data_off);
        void *addr = (void *)(&buffer[classitems[i].class_data_off]);
        //printf(" %2x %2x %2x %2x \n",(int8_t)buffer[classitems[i].class_data_off],(int8_t)buffer[classitems[i].class_data_off+1],(int8_t)buffer[classitems[i].class_data_off+2],(int8_t)buffer[classitems[i].class_data_off+3]);
        int size=0;
        int static_fields_size =  read_uleb128((uint8_t *)addr,&size);//静态成员变量的个数
        addr+=size;
        int instance_fields_size = read_uleb128((uint8_t *)addr,&size);//实例成员变量个数
        addr+=size;
        int direct_methods_size = read_uleb128((uint8_t *)addr,&size); //直接函数个数
        addr+=size;
        int virtual_methods_size = read_uleb128((uint8_t *)addr,&size);// 虚函数个数
        addr+=size;

        printf("静态成员变量:%d ,实例成员变量个数%d,直接函数个数%d.虚函数个数%d\n ",static_fields_size,instance_fields_size,direct_methods_size,virtual_methods_size);

        //开始遍历static_fields_size
        for(int i=0;i<static_fields_size;i++){
            int field_idx_diff = read_uleb128((uint8_t *)addr,&size);//实例成员变量个数
            addr+=size;
            int axxessFlags    = read_uleb128((uint8_t *)addr,&size);//实例成员变量个数
            addr+=size;
            printf("static_fields field_idx_diff %d axxessFlags %d \n ",field_idx_diff,axxessFlags);
        }
        //开始遍历instance_fields_size
        for(int i=0;i<instance_fields_size;i++){
            int field_idx_diff = read_uleb128((uint8_t *)addr,&size);//实例成员变量个数
            addr+=size;
            int axxessFlags    = read_uleb128((uint8_t *)addr,&size);//实例成员变量个数
            addr+=size;
             printf("instance_fields field_idx_diff %d axxessFlags %d \n ",field_idx_diff,axxessFlags);
        }

        //开始遍历direct_methods_size
        for(int i=0;i<direct_methods_size;i++){

            int methodIdx = read_uleb128((uint8_t *)addr,&size);//实例成员变量个数
            addr+=size;
            int accessFlags    = read_uleb128((uint8_t *)addr,&size);//实例成员变量个数
            addr+=size;
            int codeOff    = read_uleb128((uint8_t *)addr,&size);//实例成员变量个数
            addr+=size;
            printf("direct_methods methodIdx %d accessFlags %d codeOff %d \n ",methodIdx,accessFlags,codeOff);
        }

        //开始遍历virtual_methods_size
        for(int i=0;i<virtual_methods_size;i++){

            int methodIdx = read_uleb128((uint8_t *)addr,&size);//实例成员变量个数
            addr+=size;
            int accessFlags    = read_uleb128((uint8_t *)addr,&size);//实例成员变量个数
            addr+=size;
            int codeOff    = read_uleb128((uint8_t *)addr,&size);//实例成员变量个数
            addr+=size;
            printf("virtual_methods methodIdx %d accessFlags %d  codeOff %d\n ",methodIdx,accessFlags,codeOff);
        }
    }

}

void init_class_def_item(){
    classitems = (class_def_item *)malloc(header->class_defs_size_*sizeof(class_def_item));
    fseek(fp,header->class_defs_off_,0);
    fread((void *)classitems,sizeof(class_def_item),header->class_defs_size_,fp);
}

```

这里我们就把方法的 offcode 找到了

接下来解析 code item

```
struct code_item 
{
    ushort                         registers_size; //本段代码使用到的寄存器数目
    ushort                         ins_size; //method 传入参数的数目
    ushort                         outs_size; //本段代码调用其它 method 时需要的参数个数
    ushort                         tries_size;//try_item 结构的个数
    uint                         debug_info_off;//偏移地址，指向本段代码的 debug 信息存放位置，是一个 debug_info_item 结构
    uint                         insns_size;
    //ushort                         insns [insns_size]; 
    //ushort                         paddding;             // optional
    //try_item                     tries [tyies_size]; // optional
    //encoded_catch_handler_list  handlers;             // optional
}

```

```
void jx_codeitem(void * addr)
{
    code_item * code = (code_item *)addr;
    printf("寄存器数量：%d 参数数量：%d 需要的外部参数：%d try个数%d debug偏移%d 代码大小%d\n",code->registers_size,code->ins_size,code->outs_size,code->tries_size,code->debug_info_off,code->insns_size);
}

```

在 insns_size 后面就是数组 code

这个就是方法的代码所在 一串二进制 而已

到这里 我们就解析 dex 解析的差不多了  其他的知识 遇到再补。

函数抽取
----

### code 代码提取

将基础篇的 class 类解析的代码改改：

把 code 的偏移 code 的大小 code 的数据 保存下来

```
void  cq(){
    for(int i=0;i<header->class_defs_size_;i++){
        printf("data_off %d \n",classitems[i].class_data_off);
        void *addr = (void *)(&buffer[classitems[i].class_data_off]);
        //printf(" %2x %2x %2x %2x \n",(int8_t)buffer[classitems[i].class_data_off],(int8_t)buffer[classitems[i].class_data_off+1],(int8_t)buffer[classitems[i].class_data_off+2],(int8_t)buffer[classitems[i].class_data_off+3]);
        int size=0;
        int static_fields_size =  read_uleb128((uint8_t *)addr,&size);//静态成员变量的个数
        addr+=size;
        int instance_fields_size = read_uleb128((uint8_t *)addr,&size);//实例成员变量个数
        addr+=size;
        int direct_methods_size = read_uleb128((uint8_t *)addr,&size); //直接函数个数
        addr+=size;
        int virtual_methods_size = read_uleb128((uint8_t *)addr,&size);// 虚函数个数
        addr+=size;

        //printf("静态成员变量:%d ,实例成员变量个数%d,直接函数个数%d.虚函数个数%d\n ",static_fields_size,instance_fields_size,direct_methods_size,virtual_methods_size);

        //开始遍历static_fields_size
        for(int i=0;i<static_fields_size;i++){
            int field_idx_diff = read_uleb128((uint8_t *)addr,&size);//实例成员变量个数
            addr+=size;
            int axxessFlags    = read_uleb128((uint8_t *)addr,&size);//实例成员变量个数
            addr+=size;
            //printf("static_fields field_idx_diff %d axxessFlags %d \n ",field_idx_diff,axxessFlags);
        }
        //开始遍历instance_fields_size
        for(int i=0;i<instance_fields_size;i++){
            int field_idx_diff = read_uleb128((uint8_t *)addr,&size);//实例成员变量个数
            addr+=size;
            int axxessFlags    = read_uleb128((uint8_t *)addr,&size);//实例成员变量个数
            addr+=size;
            //printf("instance_fields field_idx_diff %d axxessFlags %d \n ",field_idx_diff,axxessFlags);
        }

        //开始遍历direct_methods_size
        for(int i=0;i<direct_methods_size;i++){

            int methodIdx  = read_uleb128((uint8_t *)addr,&size);//实例成员变量个数
            addr+=size;
            int accessFlags= read_uleb128((uint8_t *)addr,&size);//实例成员变量个数
            addr+=size;
            int codeOff    = read_uleb128((uint8_t *)addr,&size);//实例成员变量个数
            addr+=size;
           // printf("direct_methods methodIdx %d accessFlags %d codeOff %d \n ",methodIdx,accessFlags,codeOff);
           // jx_codeitem((void *)&buffer[codeOff]);
            code_item * code = (code_item *) &buffer[codeOff];
            //code 偏移
            //code->insns_size 代码大小
            struct code_jm{
                uint32_t codeoff;
                uint32_t insns_size;
            };
            code_jm cd;
            cd.codeoff=codeOff;
            cd.insns_size=code->insns_size;

            fwrite((void *)&cd,sizeof(code_jm),1,out);
            fwrite((void *)&code->insns[0],sizeof(char),cd.insns_size,out);

            //提取完毕
            //开始清除 code
            for(int s = 0;s <insns_size;s++){
                ((char *)&code->insns[0])[s] = 0;
            }

        }

        //开始遍历virtual_methods_size
        for(int i=0;i<virtual_methods_size;i++){

            int methodIdx = read_uleb128((uint8_t *)addr,&size);//实例成员变量个数
            addr+=size;
            int accessFlags    = read_uleb128((uint8_t *)addr,&size);//实例成员变量个数
            addr+=size;
            int codeOff    = read_uleb128((uint8_t *)addr,&size);//实例成员变量个数
            addr+=size;
           // printf("virtual_methods methodIdx %d accessFlags %d  codeOff %d\n ",methodIdx,accessFlags,codeOff);
        }
    }
}

```

dex 校验方式
--------

checksum：文件校验码，使用 alder32 算法校验文件除去 magic、checksum 外余下的所有文件区域，用于检查文件错误。

signature：使用 SHA-1 算法 hash 除去 magic，checksum 和 signature 外余下的所有文件区域 ，用于唯一识别本文件 。

顺序为 先 signature 再 checksum

```
uint32_t adler32(uint8_t *pData, size_t unLen)
{
    uint32_t s1 = 1, s2 = 0;
    size_t ulIndex;
    uint32_t unCheckSum = 0 ;

    // Process each byte of the data in order
    for (ulIndex = 0; ulIndex < unLen; ++ulIndex)
    {
        s1 = (s1 + pData[ulIndex]) % MOD_ADLER;
        s2 = (s2 + s1) % MOD_ADLER;
    }

    unCheckSum = (s2 << 16) | s1;

    return unCheckSum;
}

```

SHA-1 算法:

```
#ifndef SHA1_H
#define SHA1_H

#include <stddef.h>
#include <stdint.h>

/* MBEDTLS_ERR_SHA1_HW_ACCEL_FAILED is deprecated and should not be used. */
#define MBEDTLS_ERR_SHA1_HW_ACCEL_FAILED                  -0x0035  /**< SHA-1 hardware accelerator failed */
#define MBEDTLS_ERR_SHA1_BAD_INPUT_DATA                   -0x0073  /**< SHA-1 input data was malformed. */

// #ifdef __cplusplus
// extern "C" {
// #endif

// Regular implementation
//

/**
 * \brief          The SHA-1 context structure.
 *
 * \warning        SHA-1 is considered a weak message digest and its use
 *                 constitutes a security risk. We recommend considering
 *                 stronger message digests instead.
 *
 */
typedef struct mbedtls_sha1_context
{
    uint32_t total[2];          /*!< The number of Bytes processed.  */
    uint32_t state[5];          /*!< The intermediate digest state.  */
    uint8_t buffer[64];   /*!< The data block being processed. */
}
mbedtls_sha1_context;

/**
 * \brief          This function initializes a SHA-1 context.
 *
 * \warning        SHA-1 is considered a weak message digest and its use
 *                 constitutes a security risk. We recommend considering
 *                 stronger message digests instead.
 *
 * \param ctx      The SHA-1 context to initialize.
 *                 This must not be \c NULL.
 *
 */
void mbedtls_sha1_init( mbedtls_sha1_context *ctx );

/**
 * \brief          This function clears a SHA-1 context.
 *
 * \warning        SHA-1 is considered a weak message digest and its use
 *                 constitutes a security risk. We recommend considering
 *                 stronger message digests instead.
 *
 * \param ctx      The SHA-1 context to clear. This may be \c NULL,
 *                 in which case this function does nothing. If it is
 *                 not \c NULL, it must point to an initialized
 *                 SHA-1 context.
 *
 */
void mbedtls_sha1_free( mbedtls_sha1_context *ctx );

/**
 * \brief          This function clones the state of a SHA-1 context.
 *
 * \warning        SHA-1 is considered a weak message digest and its use
 *                 constitutes a security risk. We recommend considering
 *                 stronger message digests instead.
 *
 * \param dst      The SHA-1 context to clone to. This must be initialized.
 * \param src      The SHA-1 context to clone from. This must be initialized.
 *
 */
void mbedtls_sha1_clone( mbedtls_sha1_context *dst, const mbedtls_sha1_context *src );

/**
 * \brief          This function starts a SHA-1 checksum calculation.
 *
 * \warning        SHA-1 is considered a weak message digest and its use
 *                 constitutes a security risk. We recommend considering
 *                 stronger message digests instead.
 *
 * \param ctx      The SHA-1 context to initialize. This must be initialized.
 *
 * \return         \c 0 on success.
 * \return         A negative error code on failure.
 *
 */
int mbedtls_sha1_starts_ret( mbedtls_sha1_context *ctx );

/**
 * \brief          This function feeds an input buffer into an ongoing SHA-1
 *                 checksum calculation.
 *
 * \warning        SHA-1 is considered a weak message digest and its use
 *                 constitutes a security risk. We recommend considering
 *                 stronger message digests instead.
 *
 * \param ctx      The SHA-1 context. This must be initialized
 *                 and have a hash operation started.
 * \param input    The buffer holding the input data.
 *                 This must be a readable buffer of length \p ilen Bytes.
 * \param ilen     The length of the input data \p input in Bytes.
 *
 * \return         \c 0 on success.
 * \return         A negative error code on failure.
 */
int32_t mbedtls_sha1_update_ret( mbedtls_sha1_context *ctx, const uint8_t *input, int32_t ilen );

/**
 * \brief          This function finishes the SHA-1 operation, and writes
 *                 the result to the output buffer.
 *
 * \warning        SHA-1 is considered a weak message digest and its use
 *                 constitutes a security risk. We recommend considering
 *                 stronger message digests instead.
 *
 * \param ctx      The SHA-1 context to use. This must be initialized and
 *                 have a hash operation started.
 * \param output   The SHA-1 checksum result. This must be a writable
 *                 buffer of length \c 20 Bytes.
 *
 * \return         \c 0 on success.
 * \return         A negative error code on failure.
 */
int32_t mbedtls_sha1_finish_ret( mbedtls_sha1_context *ctx, uint8_t output[20] );

/**
 * \brief          SHA-1 process data block (internal use only).
 *
 * \warning        SHA-1 is considered a weak message digest and its use
 *                 constitutes a security risk. We recommend considering
 *                 stronger message digests instead.
 *
 * \param ctx      The SHA-1 context to use. This must be initialized.
 * \param data     The data block being processed. This must be a
 *                 readable buffer of length \c 64 Bytes.
 *
 * \return         \c 0 on success.
 * \return         A negative error code on failure.
 *
 */
int32_t mbedtls_internal_sha1_process( mbedtls_sha1_context *ctx, const uint8_t data[64] );

/**
 * \brief          This function calculates the SHA-1 checksum of a buffer.
 *
 *                 The function allocates the context, performs the
 *                 calculation, and frees the context.
 *
 *                 The SHA-1 result is calculated as
 *                 output = SHA-1(input buffer).
 *
 * \warning        SHA-1 is considered a weak message digest and its use
 *                 constitutes a security risk. We recommend considering
 *                 stronger message digests instead.
 *
 * \param input    The buffer holding the input data.
 *                 This must be a readable buffer of length \p ilen Bytes.
 * \param ilen     The length of the input data \p input in Bytes.
 * \param output   The SHA-1 checksum result.
 *                 This must be a writable buffer of length \c 20 Bytes.
 *
 * \return         \c 0 on success.
 * \return         A negative error code on failure.
 *
 */
int32_t mbedtls_sha1_ret( const uint8_t *input, int32_t ilen, uint8_t output[20] );


#endif // SHA1_H

```

```
/*
 *  FIPS-180-1 compliant SHA-1 implementation
 *
 *  Copyright The Mbed TLS Contributors
 *  SPDX-License-Identifier: Apache-2.0
 *
 *  Licensed under the Apache License, Version 2.0 (the "License"); you may
 *  not use this file except in compliance with the License.
 *  You may obtain a copy of the License at
 *
 *  http://www.apache.org/licenses/LICENSE-2.0
 *
 *  Unless required by applicable law or agreed to in writing, software
 *  distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
 *  WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 *  See the License for the specific language governing permissions and
 *  limitations under the License.
 */
/*
 *  The SHA-1 standard was published by NIST in 1993.
 *
 *  http://www.itl.nist.gov/fipspubs/fip180-1.htm
 *
 *  modifier: Honrun
 */
#include <stdint.h>
#include <string.h>
#include "sha1.h"



/*
 * 32-bit integer manipulation macros (big endian)
 */
#ifndef GET_UINT32_BE
#define GET_UINT32_BE(n,b,i)                            \
{                                                       \
        (n) = ( (uint32_t) (b)[(i)    ] << 24 )             \
          | ( (uint32_t) (b)[(i) + 1] << 16 )             \
          | ( (uint32_t) (b)[(i) + 2] <<  8 )             \
          | ( (uint32_t) (b)[(i) + 3]       );            \
}
#endif

#ifndef PUT_UINT32_BE
#define PUT_UINT32_BE(n,b,i)                            \
{                                                       \
        (b)[(i)    ] = (unsigned char) ( (n) >> 24 );       \
        (b)[(i) + 1] = (unsigned char) ( (n) >> 16 );       \
        (b)[(i) + 2] = (unsigned char) ( (n) >>  8 );       \
        (b)[(i) + 3] = (unsigned char) ( (n)       );       \
}
#endif

void mbedtls_sha1_init( mbedtls_sha1_context *ctx )
{
    if( ctx == NULL )
        return;

    memset( ctx, 0, sizeof( mbedtls_sha1_context ) );
}

void mbedtls_sha1_free( mbedtls_sha1_context *ctx )
{
    if( ctx == NULL )
        return;

    memset(ctx, 0, sizeof(mbedtls_sha1_context));
}

/*
 * SHA-1 context setup
 */
int32_t mbedtls_sha1_starts_ret( mbedtls_sha1_context *ctx )
{
    ctx->total[0] = 0;
    ctx->total[1] = 0;

    ctx->state[0] = 0x67452301;
    ctx->state[1] = 0xEFCDAB89;
    ctx->state[2] = 0x98BADCFE;
    ctx->state[3] = 0x10325476;
    ctx->state[4] = 0xC3D2E1F0;

    return( 0 );
}

int32_t mbedtls_internal_sha1_process( mbedtls_sha1_context *ctx, const uint8_t data[64] )
{
    struct
    {
        uint32_t temp, W[16], A, B, C, D, E;
    } local;

    if( ctx == NULL )
        return -1;

    GET_UINT32_BE( local.W[ 0], data,  0 );
    GET_UINT32_BE( local.W[ 1], data,  4 );
    GET_UINT32_BE( local.W[ 2], data,  8 );
    GET_UINT32_BE( local.W[ 3], data, 12 );
    GET_UINT32_BE( local.W[ 4], data, 16 );
    GET_UINT32_BE( local.W[ 5], data, 20 );
    GET_UINT32_BE( local.W[ 6], data, 24 );
    GET_UINT32_BE( local.W[ 7], data, 28 );
    GET_UINT32_BE( local.W[ 8], data, 32 );
    GET_UINT32_BE( local.W[ 9], data, 36 );
    GET_UINT32_BE( local.W[10], data, 40 );
    GET_UINT32_BE( local.W[11], data, 44 );
    GET_UINT32_BE( local.W[12], data, 48 );
    GET_UINT32_BE( local.W[13], data, 52 );
    GET_UINT32_BE( local.W[14], data, 56 );
    GET_UINT32_BE( local.W[15], data, 60 );

#define S(x,n) (((x) << (n)) | (((x) & 0xFFFFFFFF) >> (32 - (n))))

#define R(t)                                                    \
    (                                                           \
                                                                local.temp = local.W[( (t) -  3 ) & 0x0F] ^             \
                                                                                                         local.W[( (t) -  8 ) & 0x0F] ^             \
                                                                                local.W[( (t) - 14 ) & 0x0F] ^             \
                                                                                local.W[  (t)        & 0x0F],              \
                                                                ( local.W[(t) & 0x0F] = S(local.temp,1) )               \
        )

#define P(a,b,c,d,e,x)                                          \
        do                                                          \
    {                                                           \
            (e) += S((a),5) + F((b),(c),(d)) + K + (x);             \
            (b) = S((b),30);                                        \
    } while( 0 )

        local.A = ctx->state[0];
    local.B = ctx->state[1];
    local.C = ctx->state[2];
    local.D = ctx->state[3];
    local.E = ctx->state[4];

#define F(x,y,z) ((z) ^ ((x) & ((y) ^ (z))))
#define K 0x5A827999

    P( local.A, local.B, local.C, local.D, local.E, local.W[0]  );
    P( local.E, local.A, local.B, local.C, local.D, local.W[1]  );
    P( local.D, local.E, local.A, local.B, local.C, local.W[2]  );
    P( local.C, local.D, local.E, local.A, local.B, local.W[3]  );
    P( local.B, local.C, local.D, local.E, local.A, local.W[4]  );
    P( local.A, local.B, local.C, local.D, local.E, local.W[5]  );
    P( local.E, local.A, local.B, local.C, local.D, local.W[6]  );
    P( local.D, local.E, local.A, local.B, local.C, local.W[7]  );
    P( local.C, local.D, local.E, local.A, local.B, local.W[8]  );
    P( local.B, local.C, local.D, local.E, local.A, local.W[9]  );
    P( local.A, local.B, local.C, local.D, local.E, local.W[10] );
    P( local.E, local.A, local.B, local.C, local.D, local.W[11] );
    P( local.D, local.E, local.A, local.B, local.C, local.W[12] );
    P( local.C, local.D, local.E, local.A, local.B, local.W[13] );
    P( local.B, local.C, local.D, local.E, local.A, local.W[14] );
    P( local.A, local.B, local.C, local.D, local.E, local.W[15] );
    P( local.E, local.A, local.B, local.C, local.D, R(16) );
    P( local.D, local.E, local.A, local.B, local.C, R(17) );
    P( local.C, local.D, local.E, local.A, local.B, R(18) );
    P( local.B, local.C, local.D, local.E, local.A, R(19) );

#undef K
#undef F

#define F(x,y,z) ((x) ^ (y) ^ (z))
#define K 0x6ED9EBA1

    P( local.A, local.B, local.C, local.D, local.E, R(20) );
    P( local.E, local.A, local.B, local.C, local.D, R(21) );
    P( local.D, local.E, local.A, local.B, local.C, R(22) );
    P( local.C, local.D, local.E, local.A, local.B, R(23) );
    P( local.B, local.C, local.D, local.E, local.A, R(24) );
    P( local.A, local.B, local.C, local.D, local.E, R(25) );
    P( local.E, local.A, local.B, local.C, local.D, R(26) );
    P( local.D, local.E, local.A, local.B, local.C, R(27) );
    P( local.C, local.D, local.E, local.A, local.B, R(28) );
    P( local.B, local.C, local.D, local.E, local.A, R(29) );
    P( local.A, local.B, local.C, local.D, local.E, R(30) );
    P( local.E, local.A, local.B, local.C, local.D, R(31) );
    P( local.D, local.E, local.A, local.B, local.C, R(32) );
    P( local.C, local.D, local.E, local.A, local.B, R(33) );
    P( local.B, local.C, local.D, local.E, local.A, R(34) );
    P( local.A, local.B, local.C, local.D, local.E, R(35) );
    P( local.E, local.A, local.B, local.C, local.D, R(36) );
    P( local.D, local.E, local.A, local.B, local.C, R(37) );
    P( local.C, local.D, local.E, local.A, local.B, R(38) );
    P( local.B, local.C, local.D, local.E, local.A, R(39) );

#undef K
#undef F

#define F(x,y,z) (((x) & (y)) | ((z) & ((x) | (y))))
#define K 0x8F1BBCDC

    P( local.A, local.B, local.C, local.D, local.E, R(40) );
    P( local.E, local.A, local.B, local.C, local.D, R(41) );
    P( local.D, local.E, local.A, local.B, local.C, R(42) );
    P( local.C, local.D, local.E, local.A, local.B, R(43) );
    P( local.B, local.C, local.D, local.E, local.A, R(44) );
    P( local.A, local.B, local.C, local.D, local.E, R(45) );
    P( local.E, local.A, local.B, local.C, local.D, R(46) );
    P( local.D, local.E, local.A, local.B, local.C, R(47) );
    P( local.C, local.D, local.E, local.A, local.B, R(48) );
    P( local.B, local.C, local.D, local.E, local.A, R(49) );
    P( local.A, local.B, local.C, local.D, local.E, R(50) );
    P( local.E, local.A, local.B, local.C, local.D, R(51) );
    P( local.D, local.E, local.A, local.B, local.C, R(52) );
    P( local.C, local.D, local.E, local.A, local.B, R(53) );
    P( local.B, local.C, local.D, local.E, local.A, R(54) );
    P( local.A, local.B, local.C, local.D, local.E, R(55) );
    P( local.E, local.A, local.B, local.C, local.D, R(56) );
    P( local.D, local.E, local.A, local.B, local.C, R(57) );
    P( local.C, local.D, local.E, local.A, local.B, R(58) );
    P( local.B, local.C, local.D, local.E, local.A, R(59) );

#undef K
#undef F

#define F(x,y,z) ((x) ^ (y) ^ (z))
#define K 0xCA62C1D6

    P( local.A, local.B, local.C, local.D, local.E, R(60) );
    P( local.E, local.A, local.B, local.C, local.D, R(61) );
    P( local.D, local.E, local.A, local.B, local.C, R(62) );
    P( local.C, local.D, local.E, local.A, local.B, R(63) );
    P( local.B, local.C, local.D, local.E, local.A, R(64) );
    P( local.A, local.B, local.C, local.D, local.E, R(65) );
    P( local.E, local.A, local.B, local.C, local.D, R(66) );
    P( local.D, local.E, local.A, local.B, local.C, R(67) );
    P( local.C, local.D, local.E, local.A, local.B, R(68) );
    P( local.B, local.C, local.D, local.E, local.A, R(69) );
    P( local.A, local.B, local.C, local.D, local.E, R(70) );
    P( local.E, local.A, local.B, local.C, local.D, R(71) );
    P( local.D, local.E, local.A, local.B, local.C, R(72) );
    P( local.C, local.D, local.E, local.A, local.B, R(73) );
    P( local.B, local.C, local.D, local.E, local.A, R(74) );
    P( local.A, local.B, local.C, local.D, local.E, R(75) );
    P( local.E, local.A, local.B, local.C, local.D, R(76) );
    P( local.D, local.E, local.A, local.B, local.C, R(77) );
    P( local.C, local.D, local.E, local.A, local.B, R(78) );
    P( local.B, local.C, local.D, local.E, local.A, R(79) );

#undef K
#undef F

    ctx->state[0] += local.A;
    ctx->state[1] += local.B;
    ctx->state[2] += local.C;
    ctx->state[3] += local.D;
    ctx->state[4] += local.E;

    /* Zeroise buffers and variables to clear sensitive data from memory. */
    memset(&local, 0, sizeof(local));

    return( 0 );
}

/*
 * SHA-1 process buffer
 */
int32_t mbedtls_sha1_update_ret( mbedtls_sha1_context *ctx, const uint8_t *input, int32_t ilen )
{
    int32_t fill;
    uint32_t left;

    if((ctx == NULL) || (input == NULL))
        return -1;

    if( ilen == 0 )
        return( 0 );

    left = ctx->total[0] & 0x3F;
    fill = 64 - left;

    ctx->total[0] += (uint32_t) ilen;
    ctx->total[0] &= 0xFFFFFFFF;

    if( ctx->total[0] < (uint32_t) ilen )
        ctx->total[1]++;

    if( left && ilen >= fill )
    {
        memcpy( (void *) (ctx->buffer + left), input, fill );

        if( ( mbedtls_internal_sha1_process( ctx, ctx->buffer ) ) != 0 )
            return -2;

        input += fill;
        ilen  -= fill;
        left = 0;
    }

    while( ilen >= 64 )
    {
        if( ( mbedtls_internal_sha1_process( ctx, input ) ) != 0 )
            return -3;

        input += 64;
        ilen  -= 64;
    }

    if( ilen > 0 )
        memcpy( (void *) (ctx->buffer + left), input, ilen );

    return( 0 );
}


/*
 * SHA-1 final digest
 */
int32_t mbedtls_sha1_finish_ret( mbedtls_sha1_context *ctx, uint8_t output[20] )
{
    int32_t used;
    uint32_t high, low;

    if((ctx == NULL) || (output == NULL))
        return -1;

    /*
     * Add padding: 0x80 then 0x00 until 8 bytes remain for the length
     */
    used = ctx->total[0] & 0x3F;

    ctx->buffer[used++] = 0x80;

    if( used <= 56 )
    {
        /* Enough room for padding + length in current block */
        memset( ctx->buffer + used, 0, 56 - used );
    }
    else
    {
        /* We'll need an extra block */
        memset( ctx->buffer + used, 0, 64 - used );

        if( ( mbedtls_internal_sha1_process( ctx, ctx->buffer ) ) != 0 )
            return -2;

        memset( ctx->buffer, 0, 56 );
    }

    /*
     * Add message length
     */
    high = ( ctx->total[0] >> 29 )
           | ( ctx->total[1] <<  3 );
    low  = ( ctx->total[0] <<  3 );

    PUT_UINT32_BE( high, ctx->buffer, 56 );
    PUT_UINT32_BE( low,  ctx->buffer, 60 );

    if( ( mbedtls_internal_sha1_process( ctx, ctx->buffer ) ) != 0 )
        return -3;

    /*
     * Output final state
     */
    PUT_UINT32_BE( ctx->state[0], output,  0 );
    PUT_UINT32_BE( ctx->state[1], output,  4 );
    PUT_UINT32_BE( ctx->state[2], output,  8 );
    PUT_UINT32_BE( ctx->state[3], output, 12 );
    PUT_UINT32_BE( ctx->state[4], output, 16 );

    return( 0 );
}


int32_t mbedtls_sha1_ret( const uint8_t *input, int32_t ilen, uint8_t output[20] )
{
    if((input == NULL) || (output == NULL))
        return -1;

    mbedtls_sha1_context ctx = {0};

    mbedtls_sha1_init( &ctx );

    if( ( mbedtls_sha1_starts_ret( &ctx ) ) != 0 )
        return -2;

    if( ( mbedtls_sha1_update_ret( &ctx, input, ilen ) ) != 0 )
        return -3;

    if( ( mbedtls_sha1_finish_ret( &ctx, output ) ) != 0 )
        return -4;

    return 0;
}

```

在函数抽取后 重新计算 dex 的校验

```
     //计算signature
     uint8_t sig[20];//
     cout<<"ret返回值："<<mbedtls_sha1_ret((const uint8_t *)&(((Header *)(&buffer[0]))->file_size_),header->file_size_-32,sig);


     cout<<endl;
     for(int i=0;i<20;i++){
         ((Header *)(&buffer[0]))->signature_[i]=sig[i];
         printf("%2x",sig[i]);
     }
     cout<<endl;

     //计算chekunm
     uint32_t chekunm =  adler32((uint8_t *)&((Header *)(&buffer[0]))->signature_[0],header->file_size_-12);

     ((Header *)(&buffer[0]))->checksum_ = chekunm ;
     cout<<"cheknum:"<<hex<<chekunm<<endl;

     FILE * outPut = fopen((out_file+".finall").c_str(),"wb");

     if(outPut !=NULL){
         log("抽取dex FILE成功！\n");
         if(fwrite((void*)&buffer[0],sizeof(char),header->file_size_,outPut)){
             log("输出完成！\n");
         }

     }

```

函数还原
----

### hook 框架 -Dobby

使用编译好的架构

cmakelist

```
add_library(DobbyLib STATIC IMPORTED)
set_target_properties(
        DobbyLib
        PROPERTIES IMPORTED_LOCATION
        ${CMAKE_CURRENT_SOURCE_DIR}/jniLibs/${CMAKE_ANDROID_ARCH_ABI}/libdobby.a

)



target_link_libraries( # Specifies the target library.
        shellloader
        DobbyLib
        # Links the target library to the log library
        # included in the NDK.
        ${log-lib})

```

将下载好的 jnilibs 与 头文件 复制到项目

现在的 dobby 好像没有这个文件了

链接：[https://share.weiyun.com/YvM29bSi](https://share.weiyun.com/YvM29bSi) 密码：calvin

这是我电脑还保存的

下面演示如何 hook 一个简单的函数

```
#include <jni.h>
#include <string>
#include "dobby.h"
#include "android/log.h"
#define LOG_TAG "calvin日志"
#define log(...) __android_log_print(ANDROID_LOG_INFO, LOG_TAG, __VA_ARGS__)

void init() __attribute__((constructor));
void hook();
char * getMsg();
extern "C" JNIEXPORT jstring JNICALL
Java_com_calvin_shellloader_MainActivity_stringFromJNI(
        JNIEnv* env,
        jobject /* this */) {
    std::string hello = getMsg();
    return env->NewStringUTF(hello.c_str());
}

char * (*oldFun)() = nullptr;
char * newFun(){
    log("hook msg: %s",oldFun());
    return "this is be hooked!";
}
char * getMsg(){
    return  "this is msg!";
}
void init(){
    log("hook开始");
    hook();
}
void hook(){
    DobbyHook((void *)&getMsg,(void *)&newFun,(void **)&oldFun);
    log("完成 hook!");
}

```

### shadowhook 使用方法

为啥我要介绍这个 hook 原因嘛 不知道为什么 我用 dobby hook libc 产生了闪退 。

而且 实践才发现 shadow hook 更简单

```
1. 在 build.gradle 中增加依赖
ShadowHook 发布在 Maven Central 上。为了使用 native 依赖项，ShadowHook 使用了从 Android Gradle Plugin 4.0+ 开始支持的 Prefab 包格式。

android {
    buildFeatures {
        prefab true
    }
}

dependencies {
    implementation 'com.bytedance.android:shadowhook:1.0.9'
}
注意：ShadowHook 使用 prefab package schema v2，它是从 Android Gradle Plugin 7.1.0 开始作为默认配置的。如果你使用的是 Android Gradle Plugin 7.1.0 之前的版本，请在 gradle.properties 中加入以下配置：

android.prefabVersion=2.0.0
2. 在 CMakeLists.txt 或 Android.mk 中增加依赖
CMakeLists.txt

find_package(shadowhook REQUIRED CONFIG)

add_library(mylib SHARED mylib.c)
target_link_libraries(mylib shadowhook::shadowhook)
Android.mk

include $(CLEAR_VARS)
LOCAL_MODULE           := mylib
LOCAL_SRC_FILES        := mylib.c
LOCAL_SHARED_LIBRARIES += shadowhook
include $(BUILD_SHARED_LIBRARY)

$(call import-module,prefab/shadowhook)
3. 指定一个或多个你需要的 ABI
android {
    defaultConfig {
        ndk {
            abiFilters 'armeabi-v7a', 'arm64-v8a'
        }
    }
}
4. 增加打包选项
如果你是在一个 SDK 工程里使用 ShadowHook，你可能需要避免把 libshadowhook.so 打包到你的 AAR 里，以免 app 工程打包时遇到重复的 libshadowhook.so 文件。

android {
    packagingOptions {
        exclude '**/libshadowhook.so'
    }
}
另一方面, 如果你是在一个 APP 工程里使用 ShadowHook，你可以需要增加一些选项，用来处理重复的 libshadowhook.so 文件引起的冲突。

android {
    packagingOptions {
        pickFirst '**/libshadowhook.so'
    }
}
5. 初始化
ShadowHook 支持两种模式（shared 模式和 unique 模式），两种模式下的 proxy 函数写法稍有不同，你可以先尝试一下 unique 模式。

import com.bytedance.shadowhook.ShadowHook;

public class MySdk {
    public static void init() {
        ShadowHook.init(new ShadowHook.ConfigBuilder()
            .setMode(ShadowHook.Mode.UNIQUE)
            .build());
    }
}

```

这个里给出使用演示：

```
#include <jni.h>
#include <string>
#include "shadowhook.h"
#include "android/log.h"
#define LOG_TAG "calvin日志"
#define log(...) __android_log_print(ANDROID_LOG_INFO, LOG_TAG, __VA_ARGS__)

char* getmsg();

void init() __attribute__((constructor));
extern "C" JNIEXPORT jstring JNICALL
Java_com_calvin_loaderhook_MainActivity_stringFromJNI(
        JNIEnv* env,
        jobject /* this */) {
    std::string hello = getmsg();
    return env->NewStringUTF(hello.c_str());
}

char * (*oldFun)() = nullptr;

void hook();

char * newFun(){
    log("hook msg: %s",oldFun());
    return "this is be hooked!";
}
char* getmsg(){
    return "this is msg!";
}


void init(){
    hook();
}

void hook() {
    log("hook begin");
    shadowhook_hook_func_addr((void *)&getmsg,(void *)&newFun,(void **)&oldFun);

    log("hook over");
}

```

自我感觉 shadowhook 稳太多了。

函数还原流程思考
--------

第一步：

让我们的 dex 能被修改

第二部：

加载我们保存的 code 并放在 map 中

第三步

hook 类被加载的方法 在这里还原类的所有方法

### 加载提取的 code

```
extern "C"
JNIEXPORT void JNICALL
Java_com_calvin_loaderhook_Load_setData(JNIEnv *env, jobject thiz, jbyteArray data) {
    // TODO: implement setData()

    long len = env->GetArrayLength(data);//获取源数组长度
    char * data_code = ConvertJByteaArrayToChars(env,data);  //转char *
    log("native setData len %ld address:%x\n",len,data_code);
    struct code_jm{
        uint32_t codeoff;
        uint32_t insns_size;
    };
    long offset=0;
    while(len>0){
        code_jm * code= (code_jm *)(data_code+offset);
        log(" codeoff:%d size:%d data:%x \n",code->codeoff,code->insns_size,((char *)data_code+code->insns_size+sizeof(code_jm)+offset));

        offset =offset +sizeof(code_jm)+code->insns_size;
        len = len -(sizeof(code_jm)+code->insns_size);

        detial * data = new detial();
        data->size = code->insns_size;
        data->offset = (data_code+code->insns_size+sizeof(code_jm));

        maps.insert(pair<int,char*>(code->codeoff,(char *)data));
    }


}

```

### hook mmap

hook mmap 使加载的 dex 能被修改

我们先看看安卓源码的代码

```
void* mmap(void* addr, size_t size, int prot, int flags, int fd, off_t offset) {
76    return mmap64(addr, size, prot, flags, fd, static_cast<off64_t>(offset));
77  }
78

```

```
void hook() {
    log("hook begin");
    //shadowhook_hook_func_addr((void *)&getmsg,(void *)&newFun,(void **)&oldFun);
    shadowhook_hook_sym_name("libc.so","mmap64",(void *)&new_mmap,(void ** )&old_mmap);
    log("hook over");
}
void* (*old_mmap)(void* addr, size_t size, int prot, int flags, int fd, off64_t offset) = nullptr;

void* new_mmap(void* addr, size_t size, int __prot, int flags, int fd, off_t offset){
    int hasRead = (__prot & PROT_READ) == PROT_READ;
    int hasWrite = (__prot & PROT_WRITE) == PROT_WRITE;
    int prot = __prot;

    if(hasRead && !hasWrite) {
        prot = prot | PROT_WRITE;
       // log("fake_mmap call fd = %p,size = %d, prot = %d,flag = %d",fd,size, prot,flags);
    }

    void * num=(old_mmap)(addr,size,prot,flags,fd,offset);

    if((long)num < 0){
        num=(old_mmap)(addr,size,__prot,flags,fd,offset);
        log("fake_mmap call fd = %p,size = %d, prot = %d,flag = %d",fd,size, prot,flags);
        log("return num %lx",num);
    }
    return num;
}

```

这里我做了一个判断 如果 hook 的 mmap 失败 就不修改参数

### 加载未还原的 dex

这里我的知识点我放在 一代加固的帖子上

```
  private void ReplaceClassLoader(Context context, File[] dexFiles) {
        ClassLoader classLoader = context.getClassLoader();
        Class<? extends ClassLoader> aClass = classLoader.getClass();
        try {
            // 本程序的dexElements 等会把咱们加载的 放进去
            Object pathList = getField(aClass, "pathList").get(classLoader);
            Object[] dexElements = (Object[]) getField(pathList.getClass(), "dexElements").get(pathList);

            int lenNewDexElementsList = dexElements.length;
            List<Object[]> newDexElementsList = new ArrayList<>();
            newDexElementsList.add(dexElements);

            for (File dexFile : dexFiles) {

                File odex = getDir("odex", Context.MODE_PRIVATE);

                DexClassLoader dexClassLoader = new DexClassLoader(dexFile.getAbsolutePath(), odex.getAbsolutePath(), null, (PathClassLoader) classLoader);

                Object newPathList = getField(aClass, "pathList").get(dexClassLoader);
                Object[] newDexElements = (Object[]) getField(pathList.getClass(), "dexElements").get(newPathList);

                lenNewDexElementsList += newDexElements.length;
                newDexElementsList.add(newDexElements);
            }

            Class<?> elementClazz = dexElements.getClass().getComponentType();
            Object newDexElements = Array.newInstance(elementClazz, lenNewDexElementsList);

            int copiedLen = 0;
            for (Object[] objects : newDexElementsList) {
                Log.d(TAG, "ReplaceClassLoader: " + objects);
                System.arraycopy(objects, 0, newDexElements, copiedLen, objects.length);
                copiedLen += objects.length;
            }

            getField(pathList.getClass(), "dexElements").set(pathList, newDexElements);

        } catch (Exception e) {
            Log.d(TAG, "ReplaceClassLoader: ERROR" + e);
        }
    }

    private Field getField(Class clazz, String fieldName) {

        Field field;
        while (clazz != null) {
            try {
                field = clazz.getDeclaredField(fieldName);
                field.setAccessible(true);
                return field;
            } catch (NoSuchFieldException e) {
                clazz = clazz.getSuperclass();
            }
        }
        return null;
    }

```

### hook loadmethod

```
struct detial{
    uint32_t size;
    char * offset;
};
static map<int,char*> maps;
void (*old_LoadMethod)(void * dex_file,
                       void * method,
                void * klass,
                void* dst,
                       void * arg1,
                       void * arg2);

void new_LoadMethod(void * dex_file,
                             void * method,
                             void * klass,
                             void* dst,
                    void * arg1,
                    void * arg2
                             ){

    art::DexFile * file2 =(art::DexFile *)method;

    art::ClassAccessor::Method * met3 =(art::ClassAccessor::Method * )klass;
    if(strstr(file2->location_.c_str(),".dex")&& strstr(file2->location_.c_str(),"base.apk")){
        auto it =maps.find(met3->code_off_);
        if ( it == maps.end() ) {
            //log("未找到 off");
        }
        else {
            // 找到元素
            log("找到 off %x  ",met3->code_off_);
            log("this file name %s",file2->location_.c_str());
            log("this file base %x size %x",file2->begin_,file2->size_);
            int res = mprotect((void*)file2->begin_, (file2->size_/PAGE_SIZE +1)*PAGE_SIZE, PROT_READ|PROT_WRITE|PROT_EXEC);

            if(res ==0){
                log("权限申请成功！");
            }

            void * dest =(void *)(file2->begin_ + met3->code_off_+16);
            void * src =((detial*)it->second)->offset;
            memcpy((void *)(file2->begin_ + met3->code_off_+16), ((detial*)it->second)->offset, ((detial*)it->second)->size * 2);
        }
    }



    old_LoadMethod(dex_file,method,klass,dst,arg1,arg2);

}

```

这里有一个坑 我发现源码里面的函数是这样的

```
3771  void ClassLinker::LoadMethod(const DexFile& dex_file,
3772                               const ClassAccessor::Method& method,
3773                               Handle<mirror::Class> klass,
3774                               ArtMethod* dst)

```

如果按照源码的函数进行 hook 的话 会闪退

应该是这样的

```
3771  void ClassLinker::LoadMethod(void * it,
                                    const DexFile& dex_file,
3772                               const ClassAccessor::Method& method,
3773                               Handle<mirror::Class> klass,
3774                               ArtMethod* dst)

```

再多加一个参数就行了

这个 hook 函数主要的工作就是

1. 查找是否存在被抽空的 code off 在 map 里

2. 申请写权限

3. 拷贝还原 code

源码分析为啥 hook loadmethod
----------------------

这里我们要明确一个需求 --- 我们要代码在执行前才被还原

那么 我们就在  类 在加载的时候 还原即可

先从 findClass 开始

```
 protected Class<?> findClass(String name) throws ClassNotFoundException {
204          // First, check whether the class is present in our shared libraries.
205          if (sharedLibraryLoaders != null) {
206              for (ClassLoader loader : sharedLibraryLoaders) {
207                  try {
208                      return loader.loadClass(name);
209                  } catch (ClassNotFoundException ignored) {
210                  }
211              }
212          }
213          // Check whether the class in question is present in the dexPath that
214          // this classloader operates on.
215          List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
216          Class c = pathList.findClass(name, suppressedExceptions);
217          if (c == null) {
218              ClassNotFoundException cnfe = new ClassNotFoundException(
219                      "Didn't find class \"" + name + "\" on path: " + pathList);
220              for (Throwable t : suppressedExceptions) {
221                  cnfe.addSuppressed(t);
222              }
223              throw cnfe;
224          }
225          return c;
226      }

```

pathList.findClass

```
   public Class<?> findClass(String name, List<Throwable> suppressed) {
531          for (Element element : dexElements) {
532              Class<?> clazz = element.findClass(name, definingContext, suppressed);
533              if (clazz != null) {
534                  return clazz;
535              }
536          }
537  
538          if (dexElementsSuppressedExceptions != null) {
539              suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
540          }
541          return null;
542      }

```

element.findClass

```
 public Class<?> findClass(String name, ClassLoader definingContext,
771                  List<Throwable> suppressed) {
772              return dexFile != null ? dexFile.loadClassBinaryName(name, definingContext, suppressed)
773                      : null;
774          }

```

dexFile.loadClassBinaryName

```
@UnsupportedAppUsage
290      public Class loadClassBinaryName(String name, ClassLoader loader, List<Throwable> suppressed) {
291          return defineClass(name, loader, mCookie, this, suppressed);
292      }
293  
294      private static Class defineClass(String name, ClassLoader loader, Object cookie,
295                                       DexFile dexFile, List<Throwable> suppressed) {
296          Class result = null;
297          try {
298              result = defineClassNative(name, loader, cookie, dexFile);
299          } catch (NoClassDefFoundError e) {
300              if (suppressed != null) {
301                  suppressed.add(e);
302              }
303          } catch (ClassNotFoundException e) {
304              if (suppressed != null) {
305                  suppressed.add(e);
306              }
307          }
308          return result;
309      }

```

defineClassNative

```
private static native Class defineClassNative(String name, ClassLoader loader, Object cookie,
431                                                    DexFile dexFile)
432              throws ClassNotFoundException, NoClassDefFoundError;

```

进入 native 层

```
static jclass DexFile_defineClassNative(JNIEnv* env,
399                                          jclass,
400                                          jstring javaName,
401                                          jobject javaLoader,
402                                          jobject cookie,
403                                          jobject dexFile) {
404    std::vector<const DexFile*> dex_files;
405    const OatFile* oat_file;
406    if (!ConvertJavaArrayToDexFiles(env, cookie, /*out*/ dex_files, /*out*/ oat_file)) {
407      VLOG(class_linker) << "Failed to find dex_file";
408      DCHECK(env->ExceptionCheck());
409      return nullptr;
410    }
411  
412    ScopedUtfChars class_name(env, javaName);
413    if (class_name.c_str() == nullptr) {
414      VLOG(class_linker) << "Failed to find class_name";
415      return nullptr;
416    }
417    const std::string descriptor(DotToDescriptor(class_name.c_str()));
418    const size_t hash(ComputeModifiedUtf8Hash(descriptor.c_str()));
419    for (auto& dex_file : dex_files) {
420      const dex::ClassDef* dex_class_def =
421          OatDexFile::FindClassDef(*dex_file, descriptor.c_str(), hash);
422      if (dex_class_def != nullptr) {
423        ScopedObjectAccess soa(env);
424        ClassLinker* class_linker = Runtime::Current()->GetClassLinker();
425        StackHandleScope<1> hs(soa.Self());
426        Handle<mirror::ClassLoader> class_loader(
427            hs.NewHandle(soa.Decode<mirror::ClassLoader>(javaLoader)));
428        ObjPtr<mirror::DexCache> dex_cache =
429            class_linker->RegisterDexFile(*dex_file, class_loader.Get());
430        if (dex_cache == nullptr) {
431          // OOME or InternalError (dexFile already registered with a different class loader).
432          soa.Self()->AssertPendingException();
433          return nullptr;
434        }
435        ObjPtr<mirror::Class> result = class_linker->DefineClass(soa.Self(),
436                                                                 descriptor.c_str(),
437                                                                 hash,
438                                                                 class_loader,
439                                                                 *dex_file,
440                                                                 *dex_class_def);
441        // Add the used dex file. This only required for the DexFile.loadClass API since normal
442        // class loaders already keep their dex files live.
443        class_linker->InsertDexFileInToClassLoader(soa.Decode<mirror::Object>(dexFile),
444                                                   class_loader.Get());
445        if (result != nullptr) {
446          VLOG(class_linker) << "DexFile_defineClassNative returning " << result
447                             << " for " << class_name.c_str();
448          return soa.AddLocalReference<jclass>(result);
449        }
450      }
451    }
452    VLOG(class_linker) << "Failed to find dex_class_def " << class_name.c_str();
453    return nullptr;
454  }

```

class_linker->DefineClass

```
3057  ObjPtr<mirror::Class> ClassLinker::DefineClass(Thread* self,
3058                                                 const char* descriptor,
3059                                                 size_t hash,
3060                                                 Handle<mirror::ClassLoader> class_loader,
3061                                                 const DexFile& dex_file,
3062                                                 const dex::ClassDef& dex_class_def) {
3063    ScopedDefiningClass sdc(self);
3064    StackHandleScope<3> hs(self);
3065    metrics::AutoTimer timer{GetMetrics()->ClassLoadingTotalTime()};
3066    auto klass = hs.NewHandle<mirror::Class>(nullptr);
3067  
3068    // Load the class from the dex file.
3069    if (UNLIKELY(!init_done_)) {
3070      // finish up init of hand crafted class_roots_
3071      if (strcmp(descriptor, "Ljava/lang/Object;") == 0) {
3072        klass.Assign(GetClassRoot<mirror::Object>(this));
3073      } else if (strcmp(descriptor, "Ljava/lang/Class;") == 0) {
3074        klass.Assign(GetClassRoot<mirror::Class>(this));
3075      } else if (strcmp(descriptor, "Ljava/lang/String;") == 0) {
3076        klass.Assign(GetClassRoot<mirror::String>(this));
3077      } else if (strcmp(descriptor, "Ljava/lang/ref/Reference;") == 0) {
3078        klass.Assign(GetClassRoot<mirror::Reference>(this));
3079      } else if (strcmp(descriptor, "Ljava/lang/DexCache;") == 0) {
3080        klass.Assign(GetClassRoot<mirror::DexCache>(this));
3081      } else if (strcmp(descriptor, "Ldalvik/system/ClassExt;") == 0) {
3082        klass.Assign(GetClassRoot<mirror::ClassExt>(this));
3083      }
3084    }
3085  
3086    // For AOT-compilation of an app, we may use a shortened boot class path that excludes
3087    // some runtime modules. Prevent definition of classes in app class loader that could clash
3088    // with these modules as these classes could be resolved differently during execution.
3089    if (class_loader != nullptr &&
3090        Runtime::Current()->IsAotCompiler() &&
3091        IsUpdatableBootClassPathDescriptor(descriptor)) {
3092      ObjPtr<mirror::Throwable> pre_allocated =
3093          Runtime::Current()->GetPreAllocatedNoClassDefFoundError();
3094      self->SetException(pre_allocated);
3095      return sdc.Finish(nullptr);
3096    }
3097  
3098    // For AOT-compilation of an app, we may use only a public SDK to resolve symbols. If the SDK
3099    // checks are configured (a non null SdkChecker) and the descriptor is not in the provided
3100    // public class path then we prevent the definition of the class.
3101    //
3102    // NOTE that we only do the checks for the boot classpath APIs. Anything else, like the app
3103    // classpath is not checked.
3104    if (class_loader == nullptr &&
3105        Runtime::Current()->IsAotCompiler() &&
3106        DenyAccessBasedOnPublicSdk(descriptor)) {
3107      ObjPtr<mirror::Throwable> pre_allocated =
3108          Runtime::Current()->GetPreAllocatedNoClassDefFoundError();
3109      self->SetException(pre_allocated);
3110      return sdc.Finish(nullptr);
3111    }
3112  
3113    // This is to prevent the calls to ClassLoad and ClassPrepare which can cause java/user-supplied
3114    // code to be executed. We put it up here so we can avoid all the allocations associated with
3115    // creating the class. This can happen with (eg) jit threads.
3116    if (!self->CanLoadClasses()) {
3117      // Make sure we don't try to load anything, potentially causing an infinite loop.
3118      ObjPtr<mirror::Throwable> pre_allocated =
3119          Runtime::Current()->GetPreAllocatedNoClassDefFoundError();
3120      self->SetException(pre_allocated);
3121      return sdc.Finish(nullptr);
3122    }
3123  
3124    if (klass == nullptr) {
3125      // Allocate a class with the status of not ready.
3126      // Interface object should get the right size here. Regular class will
3127      // figure out the right size later and be replaced with one of the right
3128      // size when the class becomes resolved.
3129      if (CanAllocClass()) {
3130        klass.Assign(AllocClass(self, SizeOfClassWithoutEmbeddedTables(dex_file, dex_class_def)));
3131      } else {
3132        return sdc.Finish(nullptr);
3133      }
3134    }
3135    if (UNLIKELY(klass == nullptr)) {
3136      self->AssertPendingOOMException();
3137      return sdc.Finish(nullptr);
3138    }
3139    // Get the real dex file. This will return the input if there aren't any callbacks or they do
3140    // nothing.
3141    DexFile const* new_dex_file = nullptr;
3142    dex::ClassDef const* new_class_def = nullptr;
3143    // TODO We should ideally figure out some way to move this after we get a lock on the klass so it
3144    // will only be called once.
3145    Runtime::Current()->GetRuntimeCallbacks()->ClassPreDefine(descriptor,
3146                                                              klass,
3147                                                              class_loader,
3148                                                              dex_file,
3149                                                              dex_class_def,
3150                                                              &new_dex_file,
3151                                                              &new_class_def);
3152    // Check to see if an exception happened during runtime callbacks. Return if so.
3153    if (self->IsExceptionPending()) {
3154      return sdc.Finish(nullptr);
3155    }
3156    ObjPtr<mirror::DexCache> dex_cache = RegisterDexFile(*new_dex_file, class_loader.Get());
3157    if (dex_cache == nullptr) {
3158      self->AssertPendingException();
3159      return sdc.Finish(nullptr);
3160    }
3161    klass->SetDexCache(dex_cache);
3162    SetupClass(*new_dex_file, *new_class_def, klass, class_loader.Get());
3163  
3164    // Mark the string class by setting its access flag.
3165    if (UNLIKELY(!init_done_)) {
3166      if (strcmp(descriptor, "Ljava/lang/String;") == 0) {
3167        klass->SetStringClass();
3168      }
3169    }
3170  
3171    ObjectLock<mirror::Class> lock(self, klass);
3172    klass->SetClinitThreadId(self->GetTid());
3173    // Make sure we have a valid empty iftable even if there are errors.
3174    klass->SetIfTable(GetClassRoot<mirror::Object>(this)->GetIfTable());
3175  
3176    // Add the newly loaded class to the loaded classes table.
3177    ObjPtr<mirror::Class> existing = InsertClass(descriptor, klass.Get(), hash);
3178    if (existing != nullptr) {
3179      // We failed to insert because we raced with another thread. Calling EnsureResolved may cause
3180      // this thread to block.
3181      return sdc.Finish(EnsureResolved(self, descriptor, existing));
3182    }
3183  
3184    // Load the fields and other things after we are inserted in the table. This is so that we don't
3185    // end up allocating unfree-able linear alloc resources and then lose the race condition. The
3186    // other reason is that the field roots are only visited from the class table. So we need to be
3187    // inserted before we allocate / fill in these fields.
3188    LoadClass(self, *new_dex_file, *new_class_def, klass);
3189    if (self->IsExceptionPending()) {
3190      VLOG(class_linker) << self->GetException()->Dump();
3191      // An exception occured during load, set status to erroneous while holding klass' lock in case
3192      // notification is necessary.
3193      if (!klass->IsErroneous()) {
3194        mirror::Class::SetStatus(klass, ClassStatus::kErrorUnresolved, self);
3195      }
3196      return sdc.Finish(nullptr);
3197    }
3198  
3199    // Finish loading (if necessary) by finding parents
3200    CHECK(!klass->IsLoaded());
3201    if (!LoadSuperAndInterfaces(klass, *new_dex_file)) {
3202      // Loading failed.
3203      if (!klass->IsErroneous()) {
3204        mirror::Class::SetStatus(klass, ClassStatus::kErrorUnresolved, self);
3205      }
3206      return sdc.Finish(nullptr);
3207    }
3208    CHECK(klass->IsLoaded());
3209  
3210    // At this point the class is loaded. Publish a ClassLoad event.
3211    // Note: this may be a temporary class. It is a listener's responsibility to handle this.
3212    Runtime::Current()->GetRuntimeCallbacks()->ClassLoad(klass);
3213  
3214    // Link the class (if necessary)
3215    CHECK(!klass->IsResolved());
3216    // TODO: Use fast jobjects?
3217    auto interfaces = hs.NewHandle<mirror::ObjectArray<mirror::Class>>(nullptr);
3218  
3219    MutableHandle<mirror::Class> h_new_class = hs.NewHandle<mirror::Class>(nullptr);
3220    if (!LinkClass(self, descriptor, klass, interfaces, &h_new_class)) {
3221      // Linking failed.
3222      if (!klass->IsErroneous()) {
3223        mirror::Class::SetStatus(klass, ClassStatus::kErrorUnresolved, self);
3224      }
3225      return sdc.Finish(nullptr);
3226    }
3227    self->AssertNoPendingException();
3228    CHECK(h_new_class != nullptr) << descriptor;
3229    CHECK(h_new_class->IsResolved() && !h_new_class->IsErroneousResolved()) << descriptor;
3230  
3231    // Instrumentation may have updated entrypoints for all methods of all
3232    // classes. However it could not update methods of this class while we
3233    // were loading it. Now the class is resolved, we can update entrypoints
3234    // as required by instrumentation.
3235    if (Runtime::Current()->GetInstrumentation()->AreExitStubsInstalled()) {
3236      // We must be in the kRunnable state to prevent instrumentation from
3237      // suspending all threads to update entrypoints while we are doing it
3238      // for this class.
3239      DCHECK_EQ(self->GetState(), kRunnable);
3240      Runtime::Current()->GetInstrumentation()->InstallStubsForClass(h_new_class.Get());
3241    }
3242  
3243    /*
3244     * We send CLASS_PREPARE events to the debugger from here.  The
3245     * definition of "preparation" is creating the static fields for a
3246     * class and initializing them to the standard default values, but not
3247     * executing any code (that comes later, during "initialization").
3248     *
3249     * We did the static preparation in LinkClass.
3250     *
3251     * The class has been prepared and resolved but possibly not yet verified
3252     * at this point.
3253     */
3254    Runtime::Current()->GetRuntimeCallbacks()->ClassPrepare(klass, h_new_class);
3255  
3256    // Notify native debugger of the new class and its layout.
3257    jit::Jit::NewTypeLoadedIfUsingJit(h_new_class.Get());
3258  
3259    return sdc.Finish(h_new_class);
3260  }

```

LoadClass

```
3649  void ClassLinker::LoadClass(Thread* self,
3650                              const DexFile& dex_file,
3651                              const dex::ClassDef& dex_class_def,
3652                              Handle<mirror::Class> klass) {
3653    ClassAccessor accessor(dex_file,
3654                           dex_class_def,
3655                           /* parse_hiddenapi_class_data= */ klass->IsBootStrapClassLoaded());
3656    if (!accessor.HasClassData()) {
3657      return;
3658    }
3659    Runtime* const runtime = Runtime::Current();
3660    {
3661      // Note: We cannot have thread suspension until the field and method arrays are setup or else
3662      // Class::VisitFieldRoots may miss some fields or methods.
3663      ScopedAssertNoThreadSuspension nts(__FUNCTION__);
3664      // Load static fields.
3665      // We allow duplicate definitions of the same field in a class_data_item
3666      // but ignore the repeated indexes here, b/21868015.
3667      LinearAlloc* const allocator = GetAllocatorForClassLoader(klass->GetClassLoader());
3668      LengthPrefixedArray<ArtField>* sfields = AllocArtFieldArray(self,
3669                                                                  allocator,
3670                                                                  accessor.NumStaticFields());
3671      LengthPrefixedArray<ArtField>* ifields = AllocArtFieldArray(self,
3672                                                                  allocator,
3673                                                                  accessor.NumInstanceFields());
3674      size_t num_sfields = 0u;
3675      size_t num_ifields = 0u;
3676      uint32_t last_static_field_idx = 0u;
3677      uint32_t last_instance_field_idx = 0u;
3678  
3679      // Methods
3680      bool has_oat_class = false;
3681      const OatFile::OatClass oat_class = (runtime->IsStarted() && !runtime->IsAotCompiler())
3682          ? OatFile::FindOatClass(dex_file, klass->GetDexClassDefIndex(), &has_oat_class)
3683          : OatFile::OatClass::Invalid();
3684      const OatFile::OatClass* oat_class_ptr = has_oat_class ? &oat_class : nullptr;
3685      klass->SetMethodsPtr(
3686          AllocArtMethodArray(self, allocator, accessor.NumMethods()),
3687          accessor.NumDirectMethods(),
3688          accessor.NumVirtualMethods());
3689      size_t class_def_method_index = 0;
3690      uint32_t last_dex_method_index = dex::kDexNoIndex;
3691      size_t last_class_def_method_index = 0;
3692  
3693      // Use the visitor since the ranged based loops are bit slower from seeking. Seeking to the
3694      // methods needs to decode all of the fields.
3695      accessor.VisitFieldsAndMethods([&](
3696          const ClassAccessor::Field& field) REQUIRES_SHARED(Locks::mutator_lock_) {
3697            uint32_t field_idx = field.GetIndex();
3698            DCHECK_GE(field_idx, last_static_field_idx);  // Ordering enforced by DexFileVerifier.
3699            if (num_sfields == 0 || LIKELY(field_idx > last_static_field_idx)) {
3700              LoadField(field, klass, &sfields->At(num_sfields));
3701              ++num_sfields;
3702              last_static_field_idx = field_idx;
3703            }
3704          }, [&](const ClassAccessor::Field& field) REQUIRES_SHARED(Locks::mutator_lock_) {
3705            uint32_t field_idx = field.GetIndex();
3706            DCHECK_GE(field_idx, last_instance_field_idx);  // Ordering enforced by DexFileVerifier.
3707            if (num_ifields == 0 || LIKELY(field_idx > last_instance_field_idx)) {
3708              LoadField(field, klass, &ifields->At(num_ifields));
3709              ++num_ifields;
3710              last_instance_field_idx = field_idx;
3711            }
3712          }, [&](const ClassAccessor::Method& method) REQUIRES_SHARED(Locks::mutator_lock_) {
3713            ArtMethod* art_method = klass->GetDirectMethodUnchecked(class_def_method_index,
3714                image_pointer_size_);
3715            LoadMethod(dex_file, method, klass, art_method);
3716            LinkCode(this, art_method, oat_class_ptr, class_def_method_index);
3717            uint32_t it_method_index = method.GetIndex();
3718            if (last_dex_method_index == it_method_index) {
3719              // duplicate case
3720              art_method->SetMethodIndex(last_class_def_method_index);
3721            } else {
3722              art_method->SetMethodIndex(class_def_method_index);
3723              last_dex_method_index = it_method_index;
3724              last_class_def_method_index = class_def_method_index;
3725            }
3726            ++class_def_method_index;
3727          }, [&](const ClassAccessor::Method& method) REQUIRES_SHARED(Locks::mutator_lock_) {
3728            ArtMethod* art_method = klass->GetVirtualMethodUnchecked(
3729                class_def_method_index - accessor.NumDirectMethods(),
3730                image_pointer_size_);
3731            LoadMethod(dex_file, method, klass, art_method);
3732            LinkCode(this, art_method, oat_class_ptr, class_def_method_index);
3733            ++class_def_method_index;
3734          });
3735  
3736      if (UNLIKELY(num_ifields + num_sfields != accessor.NumFields())) {
3737        LOG(WARNING) << "Duplicate fields in class " << klass->PrettyDescriptor()
3738            << " (unique static fields: " << num_sfields << "/" << accessor.NumStaticFields()
3739            << ", unique instance fields: " << num_ifields << "/" << accessor.NumInstanceFields()
3740            << ")";
3741        // NOTE: Not shrinking the over-allocated sfields/ifields, just setting size.
3742        if (sfields != nullptr) {
3743          sfields->SetSize(num_sfields);
3744        }
3745        if (ifields != nullptr) {
3746          ifields->SetSize(num_ifields);
3747        }
3748      }
3749      // Set the field arrays.
3750      klass->SetSFieldsPtr(sfields);
3751      DCHECK_EQ(klass->NumStaticFields(), num_sfields);
3752      klass->SetIFieldsPtr(ifields);
3753      DCHECK_EQ(klass->NumInstanceFields(), num_ifields);
3754    }
3755    // Ensure that the card is marked so that remembered sets pick up native roots.
3756    WriteBarrier::ForEveryFieldWrite(klass.Get());
3757    self->AllowThreadSuspension();
3758  }

```

LoadMethod

```
3771  void ClassLinker::LoadMethod(const DexFile& dex_file,
3772                               const ClassAccessor::Method& method,
3773                               Handle<mirror::Class> klass,
3774                               ArtMethod* dst) {
3775    const uint32_t dex_method_idx = method.GetIndex();
3776    const dex::MethodId& method_id = dex_file.GetMethodId(dex_method_idx);
3777    const char* method_name = dex_file.StringDataByIdx(method_id.name_idx_);
3778  
3779    ScopedAssertNoThreadSuspension ants("LoadMethod");
3780    dst->SetDexMethodIndex(dex_method_idx);
3781    dst->SetDeclaringClass(klass.Get());
3782  
3783    // Get access flags from the DexFile and set hiddenapi runtime access flags.
3784    uint32_t access_flags = method.GetAccessFlags() | hiddenapi::CreateRuntimeFlags(method);
3785  
3786    if (UNLIKELY(strcmp("finalize", method_name) == 0)) {
3787      // Set finalizable flag on declaring class.
3788      if (strcmp("V", dex_file.GetShorty(method_id.proto_idx_)) == 0) {
3789        // Void return type.
3790        if (klass->GetClassLoader() != nullptr) {  // All non-boot finalizer methods are flagged.
3791          klass->SetFinalizable();
3792        } else {
3793          std::string temp;
3794          const char* klass_descriptor = klass->GetDescriptor(&temp);
3795          // The Enum class declares a "final" finalize() method to prevent subclasses from
3796          // introducing a finalizer. We don't want to set the finalizable flag for Enum or its
3797          // subclasses, so we exclude it here.
3798          // We also want to avoid setting the flag on Object, where we know that finalize() is
3799          // empty.
3800          if (strcmp(klass_descriptor, "Ljava/lang/Object;") != 0 &&
3801              strcmp(klass_descriptor, "Ljava/lang/Enum;") != 0) {
3802            klass->SetFinalizable();
3803          }
3804        }
3805      }
3806    } else if (method_name[0] == '<') {
3807      // Fix broken access flags for initializers. Bug 11157540.
3808      bool is_init = (strcmp("<init>", method_name) == 0);
3809      bool is_clinit = !is_init && (strcmp("<clinit>", method_name) == 0);
3810      if (UNLIKELY(!is_init && !is_clinit)) {
3811        LOG(WARNING) << "Unexpected '<' at start of method name " << method_name;
3812      } else {
3813        if (UNLIKELY((access_flags & kAccConstructor) == 0)) {
3814          LOG(WARNING) << method_name << " didn't have expected constructor access flag in class "
3815              << klass->PrettyDescriptor() << " in dex file " << dex_file.GetLocation();
3816          access_flags |= kAccConstructor;
3817        }
3818      }
3819    }
3820    if (UNLIKELY((access_flags & kAccNative) != 0u)) {
3821      // Check if the native method is annotated with @FastNative or @CriticalNative.
3822      access_flags |= annotations::GetNativeMethodAnnotationAccessFlags(
3823          dex_file, dst->GetClassDef(), dex_method_idx);
3824    }
3825    dst->SetAccessFlags(access_flags);
3826    // Must be done after SetAccessFlags since IsAbstract depends on it.
3827    if (klass->IsInterface() && dst->IsAbstract()) {
3828      dst->CalculateAndSetImtIndex();
3829    }
3830    if (dst->HasCodeItem()) {
3831      DCHECK_NE(method.GetCodeItemOffset(), 0u);
3832      if (Runtime::Current()->IsAotCompiler()) {
3833        dst->SetDataPtrSize(reinterpret_cast32<void*>(method.GetCodeItemOffset()), image_pointer_size_);
3834      } else {
3835        dst->SetCodeItem(dst->GetDexFile()->GetCodeItem(method.GetCodeItemOffset()));
3836      }
3837    } else {
3838      dst->SetDataPtrSize(nullptr, image_pointer_size_);
3839      DCHECK_EQ(method.GetCodeItemOffset(), 0u);
3840    }
3841  
3842    // Set optimization flags related to the shorty.
3843    const char* shorty = dst->GetShorty();
3844    bool all_parameters_are_reference = true;
3845    bool all_parameters_are_reference_or_int = true;
3846    bool return_type_is_fp = (shorty[0] == 'F' || shorty[0] == 'D');
3847  
3848    for (size_t i = 1, e = strlen(shorty); i < e; ++i) {
3849      if (shorty[i] != 'L') {
3850        all_parameters_are_reference = false;
3851        if (shorty[i] == 'F' || shorty[i] == 'D' || shorty[i] == 'J') {
3852          all_parameters_are_reference_or_int = false;
3853          break;
3854        }
3855      }
3856    }
3857  
3858    if (!dst->IsNative() && all_parameters_are_reference) {
3859      dst->SetNterpEntryPointFastPathFlag();
3860    }
3861  
3862    if (!return_type_is_fp && all_parameters_are_reference_or_int) {
3863      dst->SetNterpInvokeFastPathFlag();
3864    }
3865  }

```

流程大致如此。

在 LoadMethod 中存在 dexfile 的 base 和 size , 还有 codeoff 可以更方便的定位到我们抽取的函数。

在这里函数抽取加固的 demo 就完成了

下篇 实现批量与自动化的加固工具

[[培训]《安卓高级研修班 (网课)》月薪三万计划，掌握调试、分析还原 ollvm、vmp 的方法，定制 art 虚拟机自动化脱壳的方法](https://www.kanxue.com/book-section_list-84.htm)

最后于 12 小时前 被逆天而行编辑 ，原因：

[#基础理论](forum-161-1-117.htm) [#混淆加固](forum-161-1-121.htm) [#程序开发](forum-161-1-124.htm)

上传的附件：

*   [测试 apk.apk](javascript:void(0)) （5.51MB，8 次下载）