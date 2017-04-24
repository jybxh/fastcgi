# fastcgi
FastCGI规范

# 1. 介绍
FastCGI是对CGI的开放的扩展，它为所有因特网应用提供高性能，且没有Web服务器API的缺点（penalty）。  
本规范具有有限的（narrow）目标：从应用的视角规定FastCGI应用和支持FastCGI的Web服务器之间的接口。Web服务器的很多特性涉及FastCGI，举例来说，应用管理设施与应用到Web服务器的接口无关，因此不在这儿描述。  
本规范适用于Unix（更确切地说，适用于支持伯克利socket的POSIX系统）。本规范大半是简单的通信协议，与字节序无关，并且将扩展到其他系统。  
我们将通过与CGI/1.1的常规Unix实现的比较来介绍FastCGI。FastCGI被设计用来支持常驻（long-lived）应用进程，也就是应用服务器。那是与CGI/1.1的常规Unix实现的主要区别，后者构造应用进程，用它响应一个请求，以及让它退出。  
FastCGI进程的初始状态比CGI/1.1进程的初始状态更简洁，因为FastCGI进程开始不会连接任何东西。它没有常规的打开的文件stdin、stdout和stderr，而且它不会通过环境变量接收大量的信息。FastCGI进程的初始状态的关键部分是个正在监听的socket，通过它来接收来自Web服务器的连接。  
FastCGI进程在其正在监听的socket上收到一个连接之后，进程执行简单的协议来接收和发送数据。协议服务于两个目的。首先，协议在多个独立的 FastCGI请求间多路复用单个传输线路。这可支持能够利用事件驱动或多线程编程技术处理并发请求的应用。第二，在每个请求内部，协议在每个方向上提供 若干独立的数据流。这种方式，例如，stdout和stderr数据通过从应用到Web服务器的单个传输线路传递，而不是像CGI/1.1那样需要独立的管道。  
一个FastCGI应用扮演几个明确定义的角色中的一个。最常用的是响应器（Responder）角色，其中应用接收所有与HTTP请求相关的信息，并产生一个HTTP响应；那是CGI/1.1程序扮演的角色。第二个角色是认证器（Authorizer），其中应用接收所有与HTTP请求相关的信息，并产生一个认可/未经认可的判定。第三个角色是过滤器（Filter），其中应用接收所有与HTTP请求相关的信息，以及额外的来自存储在Web服务器上的文件的数据流，并产生"已过滤"版的数据流作为HTTP响应。框架是易扩展的，因而以后可定义更多的FastCGI。  
在本规范的其余部分，只要不致引起混淆，术语"FastCGI应用"、"应用进程"或"应用服务器"简写为"应用"。  
# 2. 初始进程状态
## 2.1 参数表
Web服务器缺省创建一个含有单个元素的参数表，该元素是应用的名字，用作可执行路径名的最后一部分。Web服务器可提供某种方式来指定不同的应用名，或更详细的参数表。  
注意，被Web服务器执行的文件可能是解释程序文件（以字符#!开头的文本文件），此情形中的应用参数表的构造在execve man页中描述。  
## 2.2 文件描述符
当应用开始执行时，Web服务器留下一个打开的文件描述符，FCGI_LISTENSOCK_FILENO。   
该描述符引用Web服务器创建的一个正在监听的socket。  
FCGI_LISTENSOCK_FILENO等于STDIN_FILENO。   
当应用开始执行时，标准的描述符STDOUT_FILENO和STDERR_FILENO被关闭。  
一个用于应用确定它是用CGI调用的还是用FastCGI调用的可靠方法是调用getpeername(FCGI_LISTENSOCK_FILENO)，对于FastCGI应用，它返回-1，并设置errno为ENOTCONN。  
Web服务器对于可靠传输的选择，Unix流式管道（AF_UNIX）或TCP/IP（AF_INET），是内含于FCGI_LISTENSOCK_FILENO socket的内部状态中的。  
## 2.3 环境变量
Web服务器可用环境变量向应用传参数。本规范定义了一个这样的变量，FCGI_WEB_SERVER_ADDRS；我们期望随着规范的发展定义更多。Web服务器可提供某种方式绑定其他环境变量，例如PATH变量。
## 2.4 其他状态
Web服务器可提供某种方式指定应用的初始进程状态的其他组件，例如进程的优先级、用户ID、组ID、根目录和工作目录。
# 3. 协议基础
## 3.1 符号（Notation）
我们用C语言符号来定义协议消息格式。  
所有的结构元素按照unsigned char类型定义和排列，这样ISO C编译器以明确的方式将它们展开，不带填充。结构中定义的第一字节第一个被传送，第二字节排第二个，依次类推。  
我们用两个约定来简化我们的定义。  
首先，当两个相邻的结构组件除了后缀“B1”和“B0”之外命名相同时，它表示这两个组件可视为估值为B1<<8 + B0的单个数字。该单个数字的名字是这些组件减去后缀的名字。这个约定归纳了一个由超过两个字节表示的数字的处理方式。
第二，我们扩展C结构（struct）来允许形式
```c
        struct {
            unsigned char mumbleLengthB1;
            unsigned char mumbleLengthB0;
            ... /* 其他东西 */
            unsigned char mumbleData[mumbleLength];
        };
```
表示一个变长结构，此处组件的长度由较早的一个或多个组件指示的值确定。
## 3.2 接受传输线路
FastCGI应用在文件描述符FCGI_LISTENSOCK_FILENO引用的socket上调用accept()来接收新的传输线路。如果accept()成功，而且也绑定了FCGI_WEB_SERVER_ADDRS环境变量，则应用立刻执行下列特殊处理：  
- FCGI_WEB_SERVER_ADDRS：值是一列有效的用于Web服务器的IP地址。  
    如果绑定了FCGI_WEB_SERVER_ADDRS，应用校验新线路的同级IP地址是否列表中的成员。如果校验失败（包括线路不是用TCP/IP传输的可能性），应用关闭线路作为响应。  
- FCGI_WEB_SERVER_ADDRS被表示成逗号分隔的IP地址列表。每个IP地址写成四个由小数点分隔的在区间[0..255]中的十进制数。所以该变量的一个合法绑定是FCGI_WEB_SERVER_ADDRS=199.170.183.28,199.170.183.71。  
应用可接受若干个并行传输线路，但不是必须的。  
## 3.3 记录
应用利用简单的协议执行来自Web服务器的请求。协议细节依赖应用的角色，但是大致说来，Web服务器首先发送参数和其他数据到应用，然后应用发送结果数据到Web服务器，最后应用向Web服务器发送一个请求完成的指示。  
通过传输线路流动的所有数据在FastCGI记录中运载。FastCGI记录实现两件事。首先，记录在多个独立的FastCGI请求间多路复用传输线路。该多路复用技术支持能够利用事件驱动或多线程编程技术处理并发请求的应用。第二，在单个请求内部，记录在每个方向上提供若干独立的数据流。这种方式，例如，stdout和stderr数据能通过从应用到Web服务器的单个传输线路传递，而不需要独立的管道。  
```c
        typedef struct {
            unsigned char version;
            unsigned char type;
            unsigned char requestIdB1;
            unsigned char requestIdB0;
            unsigned char contentLengthB1;
            unsigned char contentLengthB0;
            unsigned char paddingLength;
            unsigned char reserved;
            unsigned char contentData[contentLength];
            unsigned char paddingData[paddingLength];
        } FCGI_Record;
```
>- FastCGI记录由一个定长前缀后跟可变数量的内容和填充字节组成。记录包含七个组件：
>- version: 标识FastCGI协议版本。本规范评述（document）FCGI_VERSION_1。
>- type: 标识FastCGI记录类型，也就是记录执行的一般职能。特定记录类型和它们的功能在后面部分详细说明。
>- requestId: 标识记录所属的FastCGI请求。
>- contentLength: 记录的contentData组件的字节数。
>- paddingLength: 记录的paddingData组件的字节数。
>- contentData: 在0和65535字节之间的数据，依据记录类型进行解释。
>- paddingData: 在0和255字节之间的数据，被忽略。    

我们用不严格的C结构初始化语法来指定常量FastCGI记录。我们省略version组件，忽略填充（Padding），并且把requestId视为数字。因而{FCGI_END_REQUEST, 1, {FCGI_REQUEST_COMPLETE,0}}是个type == FCGI_END_REQUEST、requestId == 1且contentData == {FCGI_REQUEST_COMPLETE,0}的记录。  
**填充（Padding)**  
协议允许发送者填充它们发送的记录，并且要求接受者解释paddingLength并跳过paddingData。填充允许发送者为更有效地处理保持对齐的数据。X窗口系统协议上的经验显示了这种对齐方式的性能优势。  
我们建议记录被放置在八字节倍数的边界上。FCGI_Record的定长部分是八字节。  
**管理请求ID**  
Web服务器重用FastCGI请求ID；应用明了给定传输线路上的每个请求ID的当前状态。当应用收到一个记录{FCGI_BEGIN_REQUEST, R, ...}时，请求ID R变成有效的，而且当应用向Web服务器发送记录{FCGI_END_REQUEST, R, ...}时变成无效的。  
当请求ID R无效时，应用会忽略requestId == R的记录，除了刚才描述的FCGI_BEGIN_REQUEST记录。  
Web服务器尝试保持小的FastCGI请求ID。那种方式下应用能利用短数组而不是长数组或哈希表来明了请求ID的状态。应用也有每次接受一个请求的选项。这种情形下，应用只是针对当前的请求ID检查输入的requestId值。  
**记录类型的类型**  
有两种有用的分类FastCGI记录类型的方式。  
第一个区别在管理（management）记录和应用（application）记录之间。管理记录包含不特定于任何Web服务器请求的信息，例如关于应用的协议容量的信息。应用记录包含关于特定请求的信息，由requestId组件标识。  
管理记录有0值的requestId，也称为null请求ID。应用记录有非0的requestId。  
第二个区别在离散和连续记录之间。一个离散记录包含一个自己的所有数据的有意义的单元。一个流记录是stream的部分，也就是一连串流类型的0或更多非空记录（length != 0），后跟一个流类型的空记录（length == 0）。当连接流记录的多个contentData组件时，形成一个字节序列；该字节序列是流的值。因此流的值独立于它包含多少个记录或它的字节如何在非空记录间分配。  
这两种分类是独立的。在本版的FastCGI协议定义的记录类型中，所有管理记录类型也是离散记录类型，而且几乎所有应用记录类型都是流记录类型。但是三种应用记录类型是离散的，而且没有什么能防止在某些以后的协议版本中定义一个流式的管理记录类型。  
## 3.4 名-值对
FastCGI应用的很多角色需要读写可变数量的可变长度的值。所以为编码名-值对提供标准格式很有用。
FastCGI以名字长度，后跟值的长度，后跟名字，后跟值的形式传送名-值对。127字节或更少的长度能在一字节中编码，而更长的长度总是在四字节中编码：
```c
        typedef struct {
            unsigned char nameLengthB0;  /* nameLengthB0  >> 7 == 0 */
            unsigned char valueLengthB0; /* valueLengthB0 >> 7 == 0 */
            unsigned char nameData[nameLength];
            unsigned char valueData[valueLength];
        } FCGI_NameValuePair11;

        typedef struct {
            unsigned char nameLengthB0;  /* nameLengthB0  >> 7 == 0 */
            unsigned char valueLengthB3; /* valueLengthB3 >> 7 == 1 */
            unsigned char valueLengthB2;
            unsigned char valueLengthB1;
            unsigned char valueLengthB0;
            unsigned char nameData[nameLength];
            unsigned char valueData[valueLength
                    ((B3 & 0x7f) << 24) + (B2 << 16) + (B1 << 8) + B0];
        } FCGI_NameValuePair14;

        typedef struct {
            unsigned char nameLengthB3;  /* nameLengthB3  >> 7 == 1 */
            unsigned char nameLengthB2;
            unsigned char nameLengthB1;
            unsigned char nameLengthB0;
            unsigned char valueLengthB0; /* valueLengthB0 >> 7 == 0 */
            unsigned char nameData[nameLength
                    ((B3 & 0x7f) << 24) + (B2 << 16) + (B1 << 8) + B0];
            unsigned char valueData[valueLength];
        } FCGI_NameValuePair41;

        typedef struct {
            unsigned char nameLengthB3;  /* nameLengthB3  >> 7 == 1 */
            unsigned char nameLengthB2;
            unsigned char nameLengthB1;
            unsigned char nameLengthB0;
            unsigned char valueLengthB3; /* valueLengthB3 >> 7 == 1 */
            unsigned char valueLengthB2;
            unsigned char valueLengthB1;
            unsigned char valueLengthB0;
            unsigned char nameData[nameLength
                    ((B3 & 0x7f) << 24) + (B2 << 16) + (B1 << 8) + B0];
            unsigned char valueData[valueLength
                    ((B3 & 0x7f) << 24) + (B2 << 16) + (B1 << 8) + B0];
        } FCGI_NameValuePair44;
```
长度的第一字节的高位指示长度的编码方式。高位为0意味着一个字节的编码方式，1意味着四字节的编码方式。
名-值对格式允许发送者不用额外的编码方式就能传输二进制值，并且允许接收者立刻分配正确数量的内存，即使对于巨大的值。
## 3.5 关闭传输线路
Web服务器控制传输线路的生存期。当没有活动的请求时Web服务器能关闭线路。或者Web服务器也能把关闭的职权委托给应用（见FCGI_BEGIN_REQUEST）。该情形下，应用在指定的请求结束时关闭线路。
这种灵活性提供了多种应用风格。简单的应用会一次处理一个请求，并且为每个请求接受一个新的传输线路。更复杂的应用会通过一个或多个传输线路处理并发的请求，而且会长期保持传输线路为打开状态。
简单的应用通过在写入响应结束后关闭传输线路可得到重大的性能提升。Web服务器需要控制常驻线路的生命期。
当应用关闭一个线路或发现一个线路关闭了，它就初始化一个新线路。
# 4. 管理（Management）记录类型
## 4.1 FCGI_GET_VALUES, FCGI_GET_VALUES_RESULT
Web服务器能查询应用内部的具体的变量。典型地，服务器会在应用启动上执行查询以使系统配置的某些方面自动化。
应用把收到的查询作为记录{FCGI_GET_VALUES, 0, ...}。FCGI_GET_VALUES记录的contentData部分包含一系列值为空的名-值对。
应用通过发送补充了值的{FCGI_GET_VALUES_RESULT, 0, ...}记录来响应。如果应用不理解查询中包含的一个变量名，它从响应中忽略那个名字。
FCGI_GET_VALUES被设计为允许可扩充的变量集。初始集提供信息来帮助服务器执行应用和线路的管理：
>- FCGI_MAX_CONNS：该应用将接受的并发传输线路的最大值，例如"1"或"10"。
>- FCGI_MAX_REQS：该应用将接受的并发请求的最大值，例如"1"或"50"。
>- FCGI_MPXS_CONNS：如果应用不多路复用线路（也就是通过每个线路处理并发请求）则为 "0"，其他则为"1"。

应用可在任何时候收到FCGI_GET_VALUES记录。除了FastCGI库，应用的响应不能涉及应用固有的库。
##4.2 FCGI_UNKNOWN_TYPE
在本协议的未来版本中，管理记录类型集可能会增长。为了这种演变作准备，协议包含FCGI_UNKNOWN_TYPE管理记录。当应用收到无法理解的类型为T的管理记录时，它用{FCGI_UNKNOWN_TYPE, 0, {T}}响应。
FCGI_UNKNOWN_TYPE记录的contentData组件具有形式：
```c
        typedef struct {
            unsigned char type;    
            unsigned char reserved[7];
        } FCGI_UnknownTypeBody;
```
type组件是无法识别的管理记录的类型。
# 5. 应用（Application）记录类型
## 5.1 FCGI_BEGIN_REQUEST
Web服务器发送FCGI_BEGIN_REQUEST记录开始一个请求。
FCGI_BEGIN_REQUEST记录的contentData组件具有形式：
```c
        typedef struct {
            unsigned char roleB1;
            unsigned char roleB0;
            unsigned char flags;
            unsigned char reserved[5];
        } FCGI_BeginRequestBody;
```
role组件设置Web服务器期望应用扮演的角色。当前定义的角色有：
- FCGI_RESPONDER
- FCGI_AUTHORIZER
- FCGI_FILTER

角色在下面的第6章中作更详细地描述。
flags组件包含一个控制线路关闭的位：  
- flags & FCGI_KEEP_CONN：如果为0，则应用在对本次请求响应后关闭线路。如果非0，应用在对本次请求响应后不会关闭线路；Web服务器为线路保持响应性。  
## 5.2 名-值对流：FCGI_PARAMS
FCGI_PARAMS  
是流记录类型，用于从Web服务器向应用发送名-值对。名-值对被相继地沿着流发送，没有特定顺序。  
## 5.3 字节流：FCGI_STDIN, FCGI_DATA, FCGI_STDOUT, FCGI_STDERR
FCGI_STDIN  
是流记录类型，用于从Web服务器向应用发送任意数据。FCGI_DATA是另一种流记录类型，用于向应用发送额外数据。  
FCGI_STDOUT和FCGI_STDERR都是流记录类型，分别用于从应用向Web服务器发送任意数据和错误数据。  
## 5.4 FCGI_ABORT_REQUEST
Web服务器发送FCGI_ABORT_REQUEST记录来中止请求。收到{FCGI_ABORT_REQUEST, R}后，应用尽快用{FCGI_END_REQUEST, R, {FCGI_REQUEST_COMPLETE, appStatus}}响应。这是真实的来自应用的响应，而不是来自FastCGI库的低级确认。  
当HTTP客户端关闭了它的传输线路，可是受客户端委托的FastCGI请求仍在运行时，Web服务器中止该FastCGI请求。这种情况看似不太可能； 多数FastCGI请求具有很短的响应时间，同时如果客户端很慢则Web服务器提供输出缓冲。但是FastCGI应用与其他系统的通信或执行服务器端进栈 可能被延期。  
当不是通过一个传输线路多路复用请求时，Web服务器能通过关闭请求的传输线路来中止请求。但使用多路复用请求时，关闭传输线路具有不幸的结果，中止线路上的所有请求。  
## 5.5 FCGI_END_REQUEST
不论已经处理了请求，还是已经拒绝了请求，应用发送FCGI_END_REQUEST记录来终止请求。  
FCGI_END_REQUEST记录的contentData组件具有形式：
```c
        typedef struct {
            unsigned char appStatusB3;
            unsigned char appStatusB2;
            unsigned char appStatusB1;
            unsigned char appStatusB0;
            unsigned char protocolStatus;
            unsigned char reserved[3];
        } FCGI_EndRequestBody;
```
appStatus组件是应用级别的状态码。每种角色说明其appStatus的用法。  
protocolStatus组件是协议级别的状态码；可能的protocolStatus值是：
- FCGI_REQUEST_COMPLETE：请求的正常结束。
- FCGI_CANT_MPX_CONN：拒绝新请求。这发生在Web服务器通过一条线路向应用发送并发的请求时，后者被设计为每条线路每次处理一个请求。
- FCGI_OVERLOADED：拒绝新请求。这发生在应用用完某些资源时，例如数据库连接。
- FCGI_UNKNOWN_ROLE：拒绝新请求。这发生在Web服务器指定了一个应用不能识别的角色时。

# 6. 角色
## 6.1 角色协议
角色协议只包括带应用记录类型的记录。它们本质上利用流传输所有数据。  
为了让协议可靠以及简化应用编程，角色协议被设计使用近似顺序编组（nearly sequential marshalling）。在严格顺序编组的协议中，应用接收其第一个输入，然后是第二个，依次类推。直到收到全部。同样地，应用发送其第一个输出，然后是第二个，依次类推。直到发出全部。输入不是相互交叉的，输出也不是。  
对于某些FastCGI角色，顺序编组规则有太多限制，因为CGI程序能不受时限地（timing restriction）写入stdout和stderr。所以用到了FCGI_STDOUT和FCGI_STDERR的角色协议允许交叉这两个流。  
所有角色协议使用FCGI_STDERR流的方式恰是stderr在传统的应用编程中的使用方式：以易理解的方式报告应用级错误。FCGI_STDERR流的使用总是可选的。如果没有错误要报告，应用要么不发送FCGI_STDERR记录，要么发送一个0长度的FCGI_STDERR记录。  
当角色协议要求传输不同于FCGI_STDERR的流时，总是至少传输一个流类型的记录，即使流是空的。  
再次关注可靠的协议和简化的应用编程技术，角色协议被设计为近似请求-响应。在真正的请求-响应协议中，应用在发送其输出记录前接收其所有的输入记录。请求-响应协议不允许流水线技术（pipelining）。  
对于某些FastCGI角色，请求响应规则约束太强；毕竟，CGI程序不限于在开始写stdout前读取全部stdin。所以某些角色协议允许特定的可能性。首先，除了结尾的流输入，应用接收其所有输入。当开始接收结尾的流输入时，应用开始写其输出。  
当角色协议用FCGI_PARAMS传输文本值时，例如CGI程序从环境变量得到的值，其长度不包括结尾的null字节，而且它本身不包含null字节。需要提供environ(7)格式的名-值对的应用必须在名和值间插入等号，并在值后添加null字节。  
角色协议不支持CGI的未解析的（non-parsed）报头特性。FastCGI应用使用CGI报头Status和Location设置响应状态。  
## 6.2 响应器（Responder）
作为响应器的FastCGI应用具有同CGI/1.1一样的目的：它接收与HTTP请求关联的所有信息并产生HTTP响应。  
它足以解释怎样用响应器模拟CGI/1.1的每个元素：  
- 响应器应用通过FCGI_PARAMS接收来自Web服务器的CGI/1.1环境变量。  
- 接下来响应器应用通过FCGI_STDIN接收来自Web服务器的CGI/1.1 stdin数据。在收到流尾指示前，应用从该流接收最多CONTENT_LENGTH字节。（只当HTTP客户端未能提供时，例如因为客户端崩溃了，应用才收到少于CONTENT_LENGTH的字节。）  
- 响应器应用通过FCGI_STDOUT向Web服务器发送CGI/1.1 stdout数据，以及通过FCGI_STDERR发送CGI/1.1 stderr数据。应用同时发送这些，而非一个接一个。在开始写FCGI_STDOUT和FCGI_STDERR前，应用必须等待读取FCGI_PARAMS完成，但是不需要在开始写这两个流前完成从FCGI_STDIN读取。  
- 在发送其所有stdout和stderr数据后，响应器应用发送FCGI_END_REQUEST记录。应用设置protocolStatus组件为FCGI_REQUEST_COMPLETE，并设置appStatus组件为CGI程序通过exit系统调用返回的状态码。

响应器执行更新，例如实现POST方法，应该比较在FCGI_STDIN上收到的字节数和CONTENT_LENGTH，并且如果两数不等则中止更新。
## 6.3 认证器（Authorizer）
作为认证器的FastCGI应用接收所有与HTTP请求相关的信息，并产生一个认可/未经认可的判定。对于认可的判定，认证器也能把名-值对同HTTP请求相关联；当给出未经认可的判定时，认证器向HTTP客户端发送结束响应。  
由于CGI/1.1定义了与HTTP请求相关联的信息的极好的表示方式，认证器使用同样的表示法：  
- 认证器应用在FCGI_PARAMS流上接收来自Web服务器的HTTP信息，格式同响应器一样。Web服务器不会发送报头CONTENT_LENGTH、PATH_INFO、PATH_TRANSLATED和SCRIPT_NAME。  
- 认证器应用以同响应器一样的方式发送stdout和stderr数据。CGI/1.1响应状态指定对结果的处理。如果应用发送状态200（OK），Web服务器允许访问。 依赖于其配置，Web服务器可继续进行其他的访问检查，包括对其他认证器的请求。  
认证器应用的200响应可包含以Variable-为名字前缀的报头。这些报头从应用向Web服务器传送名-值对。例如，响应报头  
                   Variable-AUTH_METHOD: database lookup   
传输名为AUTH-METHOD的值"database lookup"。服务器把这样的名-值对同HTTP请求相关联，并且把它们包含在后续的CGI或FastCGI请求中，这些请求在处理HTTP请求的过程中执行。当应用给出200响应时，服务器忽略名字不以Variable-为前缀的响应报头，并且忽略任何响应内容。  
对于“200”（OK）以外的认证器响应状态值，Web服务器拒绝访问并将响应状态、报头和内容发回HTTP客户端。  
## 6.4 过滤器（Filter）
作为过滤器的FastCGI应用接收所有与HTTP请求相关联的信息，以及额外的来自存储在Web服务器上的文件的数据流，并产生数据流的“已过滤”版本作为HTTP响应。  
过滤器在功能上类似响应器，接受一个数据文件作为参数。区别是，过滤器使得数据文件和过滤器本身都能用Web服务器的访问控制机制进行访问控制，而响应器接受数据文件名作为参数，必须在数据文件上执行自己的访问控制检查。  
过滤器采取的步骤与响应器的相似。服务器首先提供环境变量，然后是标准输入（常规形式的POST数据），最后是数据文件输入：  
- 如同响应器，过滤器应用通过FCGI_PARAMS接收来自Web服务器的名-值对。过滤器应用接收两个过滤器特定的变量：FCGI_DATA_LAST_MOD和FCGI_DATA_LENGTH。  
- 接下来，过滤器应用通过FCGI_STDIN接收来自Web服务器的CGI/1.1 stdin数据。在收到流尾指示以前，应用从该流接收最多CONTENT_LENGTH字节。（只有HTTP客户端未能提供时，应用收到的才少于CONTENT_LENGTH字节，例如因为客户端崩溃了。）  
- 下一步，过滤器应用通过FCGI_DATA接收来自Web服务器的文件数据。该文件的最后修改时间（表示成自UTC 1970年1月1日以来的整秒数）是FCGI_DATA_LAST_MOD；应用可能查阅该变量并从缓存作出响应，而不读取文件数据。在收到流尾指示以前，应用从该流接收最多FCGI_DATA_LENGTH字节。  
- 过滤器应用通过FCGI_STDOUT向Web服务器发送CGI/1.1 stdout数据，以及通过FCGI_STDERR的CGI/1.1 stderr数据。应用同时发送这些，而非相继地。在开始写入FCGI_STDOUT和FCGI_STDERR以前，应用必须等待读取FCGI_STDIN完成，但是不需要在开始写入这两个流以前完成从FCGI_DATA的读取。  
- 在发送其所有的stdout和stderr数据之后，应用发送FCGI_END_REQUEST记录。应用设定protocolStatus组件为FCGI_REQUEST_COMPLETE，以及appStatus组件为类似的CGI程序通过exit系统调用返回的状态代码。  

过滤器应当把在FCGI_STDIN上收到的字节数同CONTENT_LENGTH比较，以及把FCGI_DATA上的同FCGI_DATA_LENGTH比较。如果数字不匹配且过滤器是个查询，过滤器响应应当提供数据丢失的指示。如果数字不匹配且过滤器是个更新，过滤器应当中止更新。  
# 7. 错误
FastCGI应用以0状态退出来指出它故意结束了，例如，为了执行原始形式的垃圾收集。FastCGI应用以非0状态退出被假定为崩溃了。以0或非0状态退出的Web服务器或其他的应用管理器如何响应应用超出了本规范的范围。  
Web服务器能通过向FastCGI应用发送SIGTERM来要求它退出。如果应用忽略SIGTERM，Web服务器能采用SIGKILL。  
FastCGI应用使用FCGI_STDERR流和FCGI_END_REQUEST记录的appStatus组件报告应用级别错误。在很多情形中，错误会通过FCGI_STDOUT流直接报告给用户。  
在Unix上，应用向syslog报告低级错误，包括FastCGI协议错误和FastCGI环境变量中的语法错误。依赖于错误的严重性，应用可能继续或以非0状态退出。  
# 8. 类型和常量
```c
/*
 * 正在监听的socket文件编号
 */
#define FCGI_LISTENSOCK_FILENO 0

typedef struct {
    unsigned char version;
    unsigned char type;
    unsigned char requestIdB1;
    unsigned char requestIdB0;
    unsigned char contentLengthB1;
    unsigned char contentLengthB0;
    unsigned char paddingLength;
    unsigned char reserved;
} FCGI_Header;
/*
 * FCGI_Header中的字节数。协议的未来版本不会减少该数。
 */
#define FCGI_HEADER_LEN  8
/*
 * 可用于FCGI_Header的version组件的值
 */
#define FCGI_VERSION_1           1
/*
 * 可用于FCGI_Header的type组件的值
 */
#define FCGI_BEGIN_REQUEST       1
#define FCGI_ABORT_REQUEST       2
#define FCGI_END_REQUEST         3
#define FCGI_PARAMS              4
#define FCGI_STDIN               5
#define FCGI_STDOUT              6
#define FCGI_STDERR              7
#define FCGI_DATA                8
#define FCGI_GET_VALUES          9
#define FCGI_GET_VALUES_RESULT  10
#define FCGI_UNKNOWN_TYPE       11
#define FCGI_MAXTYPE (FCGI_UNKNOWN_TYPE)
/*
 * 可用于FCGI_Header的requestId组件的值
 */
#define FCGI_NULL_REQUEST_ID     0

typedef struct {
    unsigned char roleB1;
    unsigned char roleB0;
    unsigned char flags;
    unsigned char reserved[5];
} FCGI_BeginRequestBody;

typedef struct {
    FCGI_Header header;
    FCGI_BeginRequestBody body;
} FCGI_BeginRequestRecord;
/*
 * 可用于FCGI_BeginRequestBody的flags组件的掩码
 */
#define FCGI_KEEP_CONN  1
/*
 * 可用于FCGI_BeginRequestBody的role组件的值
 */
#define FCGI_RESPONDER  1
#define FCGI_AUTHORIZER 2
#define FCGI_FILTER     3

typedef struct {
    unsigned char appStatusB3;
    unsigned char appStatusB2;
    unsigned char appStatusB1;
    unsigned char appStatusB0;
    unsigned char protocolStatus;
    unsigned char reserved[3];
} FCGI_EndRequestBody;

typedef struct {
    FCGI_Header header;
    FCGI_EndRequestBody body;
} FCGI_EndRequestRecord;
/*
 * 可用于FCGI_EndRequestBody的protocolStatus组件的值
 */
#define FCGI_REQUEST_COMPLETE 0
#define FCGI_CANT_MPX_CONN    1
#define FCGI_OVERLOADED       2
#define FCGI_UNKNOWN_ROLE     3
/*
 * 可用于FCGI_GET_VALUES/FCGI_GET_VALUES_RESULT记录的变量名
 */
#define FCGI_MAX_CONNS  "FCGI_MAX_CONNS"
#define FCGI_MAX_REQS   "FCGI_MAX_REQS"
#define FCGI_MPXS_CONNS "FCGI_MPXS_CONNS"

typedef struct {
    unsigned char type;    
    unsigned char reserved[7];
} FCGI_UnknownTypeBody;

typedef struct {
    FCGI_Header header;
    FCGI_UnknownTypeBody body;
} FCGI_UnknownTypeRecord;
```
# 9. 参考
National Center for Supercomputer Applications, The Common Gateway Interface, version CGI/1.1.  
D.R.T. Robinson, The WWW Common Gateway Interface Version 1.1, Internet-Draft, 15 February 1996.  
# A. 表：记录类型的属性
下面的图表列出了所有记录类型，并指出各自的这些属性：  
- WS->App：该类型的记录只能由Web服务器发送到应用。其他类型的记录只能由应用发送到Web服务器。  
- management：该类型的记录含有非特定于某个Web服务器请求的信息，而且使用null请求ID。其他类型的记录含有请求特定的信息，而且不能使用null请求ID。  
- stream：该类型的记录组成一个由带有空contentData的记录结束的流。其他类型的记录是离散的；各自携带一个有意义的数据单元。  
```
                                WS->App   management  stream

        FCGI_GET_VALUES           x          x
        FCGI_GET_VALUES_RESULT               x
        FCGI_UNKNOWN_TYPE                    x

        FCGI_BEGIN_REQUEST        x
        FCGI_ABORT_REQUEST        x
        FCGI_END_REQUEST
        FCGI_PARAMS               x                    x
        FCGI_STDIN                x                    x
        FCGI_DATA                 x                    x
        FCGI_STDOUT                                    x 
        FCGI_STDERR                                    x     
```

# B. 典型的协议消息流程
用于示例的补充符号约定：  
- 流记录的contentData（FCGI_PARAMS、FCGI_STDIN、FCGI_STDOUT和FCGI_STDERR）被描述成一个字符串。以" ... "结束的字符串是太长而无法显示的，所以只显示前缀。  
- 发送到Web服务器的消息相对于收自Web服务器的消息缩进排版。  
- 消息以应用经历的时间顺序显示。  

1. 在stdin上不带数据的简单请求，以及成功的响应：
```c
{FCGI_BEGIN_REQUEST,   1, {FCGI_RESPONDER, 0}}
{FCGI_PARAMS,          1, "\013\002SERVER_PORT80\013\016SERVER_ADDR199.170.183.42 ... "}
{FCGI_PARAMS,          1, ""}
{FCGI_STDIN,           1, ""}

{FCGI_STDOUT,      1, "Content-type: text/html\r\n\r\n<html>\n<head> ... "}
{FCGI_STDOUT,      1, ""}
{FCGI_END_REQUEST, 1, {0, FCGI_REQUEST_COMPLETE}}
```
2. 类似例1，但这次在stdin有数据。Web服务器选择用比之前更多的FCGI_PARAMS记录发送参数：
```c
{FCGI_BEGIN_REQUEST,   1, {FCGI_RESPONDER, 0}}
{FCGI_PARAMS,          1, "\013\002SERVER_PORT80\013\016SER"}
{FCGI_PARAMS,          1, "VER_ADDR199.170.183.42 ... "}
{FCGI_PARAMS,          1, ""}
{FCGI_STDIN,           1, "quantity=100&item=3047936"}
{FCGI_STDIN,           1, ""}

{FCGI_STDOUT,      1, "Content-type: text/html\r\n\r\n<html>\n<head> ... "}
{FCGI_STDOUT,      1, ""}
{FCGI_END_REQUEST, 1, {0, FCGI_REQUEST_COMPLETE}}
```
3. 类似例1，但这次应用发现了错误。应用把一条消息记录到stderr，向客户端返回一个页面，并且向Web服务器返回非0退出状态。应用选择用更多FCGI_STDOUT记录发送页面：
```c
{FCGI_BEGIN_REQUEST,   1, {FCGI_RESPONDER, 0}}
{FCGI_PARAMS,          1, "\013\002SERVER_PORT80\013\016SERVER_ADDR199.170.183.42 ... "}
{FCGI_PARAMS,          1, ""}
{FCGI_STDIN,           1, ""}

{FCGI_STDOUT,      1, "Content-type: text/html\r\n\r\n<ht"}
{FCGI_STDERR,      1, "config error: missing SI_UID\n"}
{FCGI_STDOUT,      1, "ml>\n<head> ... "}
{FCGI_STDOUT,      1, ""}
{FCGI_STDERR,      1, ""}
{FCGI_END_REQUEST, 1, {938, FCGI_REQUEST_COMPLETE}}
```
4. 在单条线路上多路复用的两个例1实例。第一个请求比第二个难，所以应用颠倒次序完成这些请求：
```c
{FCGI_BEGIN_REQUEST,   1, {FCGI_RESPONDER, FCGI_KEEP_CONN}}
{FCGI_PARAMS,          1, "\013\002SERVER_PORT80\013\016SERVER_ADDR199.170.183.42 ... "}
{FCGI_PARAMS,          1, ""}
{FCGI_BEGIN_REQUEST,   2, {FCGI_RESPONDER, FCGI_KEEP_CONN}}
{FCGI_PARAMS,          2, "\013\002SERVER_PORT80\013\016SERVER_ADDR199.170.183.42 ... "}
{FCGI_STDIN,           1, ""}

{FCGI_STDOUT,      1, "Content-type: text/html\r\n\r\n"}

{FCGI_PARAMS,          2, ""}
{FCGI_STDIN,           2, ""}

{FCGI_STDOUT,      2, "Content-type: text/html\r\n\r\n<html>\n<head> ... "}
{FCGI_STDOUT,      2, ""}
{FCGI_END_REQUEST, 2, {0, FCGI_REQUEST_COMPLETE}}
{FCGI_STDOUT,      1, "<html>\n<head> ... "}
{FCGI_STDOUT,      1, ""}
{FCGI_END_REQUEST, 1, {0, FCGI_REQUEST_COMPLETE}}
```
