# ESP32 通过HTTP与Web Server进行GET与POST交互

本次记录的重点主要为"记录建立通信时出现的各类报错"

## 可以正常发起GET请求, 但是得到的JSON串不正确, 比如出现乱码等等情况

**正确代码如下:**

``` C++
    int httpCode = http.GET();
    if (httpCode > 0)
    {
        if (httpCode == HTTP_CODE_OK) // HTTP请求无异常
        {
            String *payload = new String(1024);
            *payload = http.getString();
            Serial.printf("get %s\r\n", payload->c_str());
            WiFi_json_root = (char *)payload->c_str(); // 读取get到的json串
            
            cJSON *root = cJSON_Parse(WiFi_json_root);
            if (root == NULL)
            {
                Serial.printf("Error before: [%s]\n", cJSON_GetErrorPtr());
            }
            cJSON *cmd = cJSON_GetObjectItem(root, "cmd");

            switch (cmd->valueint)
            {
            case 1:
                cmd1(root);
                break;
            default:
                Serial.printf("error cmd!\r\n");
                std::string post_Payload("something error.");
                http.POST(post_Payload.c_str());
                break;
            }
```

**出错时的代码问题为:**

1. 由于使用的是arduino的http库, 所以getString之后得到的变量是String类型的, 这部分没有做好变量转换(转换成C语言的char*字符串变量), 从而导致解析JSON串时出现报错问题.

注意点:

​	如果http得到的payload很大的话, 一定要像代码中一样new出来一片内存区, 仅仅留一个指针来指向那块内存区间, 否则使用RTOS线程的话极有可能造成栈溢出.同时,在处理完payload之后, 要及时delete掉这片内存, 防止循环执行之后产生内存溢出问题.



## 正常GET, POST的时也能够正常生成JSON串, 但是服务器端无法接收到POST出去的数据

**代码如下:**

```C++
void cmd1(cJSON *root){
    cJSON *tx_root = cJSON_CreateObject();
    cJSON_AddItemToObject(tx_root, "res", cJSON_CreateNumber(0));
    cJSON_AddItemToObject(tx_root, "cmd", cJSON_CreateNumber(1));

    static int x = 0,y = 0,z = 0;

    cJSON_AddItemToObject(tx_root, "X_SPEED", cJSON_CreateNumber(x++));
    cJSON_AddItemToObject(tx_root, "Y_SPEED", cJSON_CreateNumber(y++));
    cJSON_AddItemToObject(tx_root, "Z_SPEED", cJSON_CreateNumber(z++));
    
    char* json_string = cJSON_Print(tx_root);

    int httpCode = http.connected();
    if (httpCode == true)
    {
        http.POST(String(json_string));
        // Serial.printf("send: %s\r\n", json_string);
    }
    else
    {
        TX_Characteristics.setValue("http disconnected!");
        TX_Characteristics.notify();
        Serial.printf("http disconnected!\r\n");
    }

    cJSON_Delete(tx_root);
    free(json_string);
}
```

**出错原因:**

1. http本身是短连接, 我在WiFi_setup时进行了http的连接, 并且后续虽然在读取http.connected()的情况时返回了正常连接的状态, 但是实测并无法POST出去数据, 服务端收到的数据是空的. 应当在每次进行GET与POST之前关闭上一次的连接, 并重新开启新的一次连接.

2. 对于部分服务端架构, 我们单单只进行一次http.begin()是不够的, 要加入http.addheader()函数, 来声明一个包的格式, 这样服务端才能够正确的解析数据包.即使用**http.begin()+http.addheader()**代替**单独的http.begin().** 示例代码如下

   ``` C++
       std::stringstream urlStream;
       urlStream << "http://" << WiFi_Data.serverip << ":" << WiFi_Data.serverport << "/cmd/connect";
       Serial.printf("Try to connect %s\r\n", urlStream.str().c_str());
       http.begin(*wifi, urlStream.str().c_str()); // 连接服务器对应域名
       http.addHeader("Content-Type", "application/json");	//我使用的是JSON格式的数据包, 这里就声明为JSON
   ```

   

## 正常GET与正常POST, 但是在运行时报错缓冲区的问题

**报错情况如下**

![image-20240331212109262](E:/Typora_note/photos/image-20240331212109262.png)

这一问题已经被github与Stack Overflow的大佬解决, issue连接如下

https://github.com/espressif/arduino-esp32/issues/6129#issuecomment-1418051304

![image-20240331212151907](E:/Typora_note/photos/image-20240331212151907.png)

**解决方案为:** 

​	新建了一份代码wifiFix.h与wifiFix.cpp, 并在其中重写了flush函数, 实测效果很好, 不在出现报错, 长时间运行也稳定性很好(虽然有报错的时候也不影响正常运行)



代码内容如下:

**wifiFix.h**

```c++
#ifndef __WIFIFIX_H
#define __WIFIFIX_H

#include "WiFi.h"

class WiFiClientFixed : public WiFiClient {
public:
	void flush() override;
};

#endif
```

**wifiFix.cpp**

``` c++
#include "wifiFix.h"

#define WIFI_CLIENT_FLUSH_BUFFER_SIZE    (1024)

void WiFiClientFixed::flush() {
	int res;
	size_t a = available(), toRead = 0;
	if (!a) {
		return;//nothing to flush
	}
	auto *buf = (uint8_t *) malloc(WIFI_CLIENT_FLUSH_BUFFER_SIZE);
	if (!buf) {
		return;//memory error
	}
	while (a) {
		// override broken WiFiClient flush method, ref https://github.com/espressif/arduino-esp32/issues/6129#issuecomment-1237417915
		res = read(buf, min(a, (size_t)WIFI_CLIENT_FLUSH_BUFFER_SIZE));
		if (res < 0) {
			log_e("fail on fd %d, errno: %d, \"%s\"", fd(), errno, strerror(errno));
			stop();
			break;
		}
		a -= res;
	}
	free(buf);
}
```

**加入此代码后, 用http.begin(*wifi, url);替换原有的http.begin(url)即可, 无需修改原有的WiFi配置或其他配置**

