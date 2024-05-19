# 接收与处理JSON数据串的代码模块

在前些天的示例中, 我们已经实现了利用串口中断来高效地接收JSON数据包, 并将其使用消息队列传递给解析线程

在这次的项目中, 我们将创建一个高效的Json指令处理模块



## 接收并处理JSON数据包字符串

在cJSON中, JSON字符串的参数为**(const char* value)**

所以我们应当传入一个 const char* 或 char* 的字符串

### 开始对字符串进行解析	cmd_startParse

``` c
void cmd_startParse(char* JsonString){
    cJSON *root = cJSON_Parse(JsonString);
    if(root == NULL){
        uart3_printf("Error before: [%s]\n", cJSON_GetErrorPtr());
    }
    cJSON *cmd = cJSON_GetObjectItem(root, "cmd");
    switch (cmd->valueint) {
        case 1:
            cmd_setAngularDeviation(root);
            break;
        case 2:
            cmd_setServoAngle(root);
            break;
        case 3:
            cmd_getMotorSpeed(root);
            break;
        case 4:
            cmd_getMotorAngle(root);
            break;
        case 5:
            cmd_getMotorCurrent(root);
            break;
        case 6:
            cmd_getSystemMode(root);
            break;
        case 7:
            cmd_setHeartBeat(root);
            break;
        default:
            break;
    }
    cJSON_Delete(root);
}
```

在这里, 我根据自己的需要设定了7种不同的状态处理.

为了区分不同的指令, 我的数据包格式中包含了"cmd"这一int变量, 示例如下:

> **cmd_setAngularDeviation:**
>
> {  	
> "cmd":1,  	
> "Deviation": 5  	
> }

### 进入JSON数据包的解析, 并进行后续处理

解析过程如下:

1. 创建一个cJSON对象, 将其作为本次解析的根节点

   ``` c
   	cJSON *root = cJSON_Parse(JsonString);
   	//若是数据包不符合JSON格式, 则root不会被创建, 进入以下报错
       if(root == NULL){
           uart3_printf("Error before: [%s]\n", cJSON_GetErrorPtr());
       }
   ```

   

2. 根据cmd值的不同, 进入不同的解析函数之中

   ``` c
   cJSON *cmd = cJSON_GetObjectItem(root, "cmd");
       switch (cmd->valueint) {
           case 1:
               cmd_setAngularDeviation(root);
               break;
           case 2:
               cmd_setServoAngle(root);
               break;
           case 3:
               cmd_getMotorSpeed(root);
               break;
           case 4:
               cmd_getMotorAngle(root);
               break;
           case 5:
               cmd_getMotorCurrent(root);
               break;
           case 6:
               cmd_getSystemMode(root);
               break;
           case 7:
               cmd_setHeartBeat(root);
               break;
           default:
               break;
       }
   cJSON_Delete(root);
   ```

   

3. 在解析函数中对数据包的内容, **以变量名为目标**进行针对性的解析

   ``` c
   void cmd_setServoAngle(cJSON* root){
       cJSON *cmd_AngleLeft = cJSON_GetObjectItem(root, "ServoAngle_Left");
       cJSON *cmd_AngleRight = cJSON_GetObjectItem(root, "ServoAngle_Right");
   
       uart3_printf("get left Angle: %d\tRight Angle: %d\r\n", cmd_AngleLeft->valueint, cmd_AngleRight->valueint);
   
   //    cJSON *response_root = cJSON_CreateObject();
   //    cJSON_AddItemToObject(response_root, "res", cJSON_CreateNumber(0));
   //    cJSON_AddItemToObject(response_root, "cmd", cJSON_CreateNumber(2));
   
   //    char* responseText = cJSON_Print(response_root);
   
   //    uart3_printf("%s", responseText);
   
   //    cJSON_Delete(response_root);
   //    free(responseText);
   }
   ```

   

4. 创建一个JSON数据包用来回复消息

   ``` c
   void cmd_setServoAngle(cJSON* root){
   //    cJSON *cmd_AngleLeft = cJSON_GetObjectItem(root, "ServoAngle_Left");
   //    cJSON *cmd_AngleRight = cJSON_GetObjectItem(root, "ServoAngle_Right");
   
   //    uart3_printf("get left Angle: %d\tRight Angle: %d\r\n", cmd_AngleLeft->valueint, cmd_AngleRight->valueint);
   
       cJSON *response_root = cJSON_CreateObject();
       cJSON_AddItemToObject(response_root, "res", cJSON_CreateNumber(0));
       cJSON_AddItemToObject(response_root, "cmd", cJSON_CreateNumber(2));
   
       char* responseText = cJSON_Print(response_root);	//生成完整的JSON数据串
   
       uart3_printf("%s", responseText);
   
   //    cJSON_Delete(response_root);
   //    free(responseText);
   }
   ```

   

5. 保护内存安全, 释放掉先前创建的无用的指针变量

   ``` c
   void cmd_setServoAngle(cJSON* root){
   //    cJSON *cmd_AngleLeft = cJSON_GetObjectItem(root, "ServoAngle_Left");
   //    cJSON *cmd_AngleRight = cJSON_GetObjectItem(root, "ServoAngle_Right");
   //
   //    uart3_printf("get left Angle: %d\tRight Angle: %d\r\n", cmd_AngleLeft->valueint, cmd_AngleRight->valueint);
   //
   //    cJSON *response_root = cJSON_CreateObject();
   //    cJSON_AddItemToObject(response_root, "res", cJSON_CreateNumber(0));
   //    cJSON_AddItemToObject(response_root, "cmd", cJSON_CreateNumber(2));
   //
   //    char* responseText = cJSON_Print(response_root);
   //
   //    uart3_printf("%s", responseText);
   
       cJSON_Delete(response_root);
       free(responseText);
   }
   ```

   