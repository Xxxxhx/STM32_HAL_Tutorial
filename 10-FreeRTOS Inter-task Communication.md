# FreeRTOS Inter-task Communication

## Introduction

In RTOS, only one task can be executed at one time, so inter-task communication is needed to exchange information between tasks. In regular operating systems, pipes, message queues, sockets and share memories. But in RTOS, message queues and mail queues are used most frequently, which is thread safe. 



## Message queues

Message queues in FreeRTOS can store a finite length of data with certain size, which is a FIFO(First In First Out) queue. Since it has been integrated into STM32Cube, we can configure it easily.

### Configuration in STM32Cube

Go to the **Tasks and Queues** tab, and add a queue and two tasks as follow:

![1574412947537](Inter-task%20Communication.assets/1574412947537.png)

Generate the code.

### API

Compare to the code in **freertos.c** in pervious lab, there are two more lines added in ``MX_FREERTOS_Init``:

```c
  /* Create the queue(s) */
  /* definition and creation of myQueue01 */
  osMessageQDef(myQueue01, 16, uint16_t);
  myQueue01Handle = osMessageCreate(osMessageQ(myQueue01), NULL);
```

``osMessageQDef`` is a macro. Expand it we will get:

```c
const osMessageQDef_t os_messageQ_def_myQueue01 = \
{ (16), sizeof (uint16_t), ((void *)0), ((void *)0)  }
```

``osMessageQ`` is a macro also:

```c
&os_messageQ_def_myQueue01
```

And use ``osMessageCreate`` to create a message queue with the reference returned.

```c
osMessageQId osMessageCreate (const osMessageQDef_t *queue_def, osThreadId thread_id)
```

We can use the following two functions to write and read the message queue:

```c
osStatus osMessagePut (osMessageQId queue_id, uint32_t info, uint32_t millisec)
osEvent osMessageGet (osMessageQId queue_id, uint32_t millisec)
```

Note that ``osEvent`` is a struct with two unions inside:

```c
/// Event structure contains detailed information about an event.
/// \note MUST REMAIN UNCHANGED: \b os_event shall be consistent in every CMSIS-RTOS.
///       However the struct may be extended at the end.
typedef struct  {
  osStatus                 status;     ///< status code: event or error information
  union  {
    uint32_t                    v;     ///< message as 32-bit value
    void                       *p;     ///< message or mail as void pointer
    int32_t               signals;     ///< signal flags
  } value;                             ///< event value
  union  {
    osMailQId             mail_id;     ///< mail id obtained by \ref osMailCreate
    osMessageQId       message_id;     ///< message id obtained by \ref osMessageCreate
  } def;                               ///< event definition
} osEvent;

```

Now we can use these two functions to implement ``MsgProducerTask`` and ``MsgConsumerTask``:

```c
/* USER CODE END Header_MsgProducerTask */
void MsgProducerTask(void const * argument)
{
  /* USER CODE BEGIN MsgProducerTask */
  /* Infinite loop */
  for(;;)
  {
    osDelay(1000);
    osMessagePut(myQueue01Handle, 2, osWaitForever);
    osDelay(2000);
    osMessagePut(myQueue01Handle, 4, osWaitForever);
  }
  /* USER CODE END MsgProducerTask */
}

/* USER CODE BEGIN Header_MsgConsumerTask */
/**
* @brief Function implementing the MsgConsumer thread.
* @param argument: Not used
* @retval None
*/
/* USER CODE END Header_MsgConsumerTask */
void MsgConsumerTask(void const * argument)
{
  /* USER CODE BEGIN MsgConsumerTask */
  osEvent event;
  char msg[20];
  /* Infinite loop */
  for(;;)
  {
    event = osMessageGet(myQueue01Handle, osWaitForever);
    if (event.status == osEventMessage)
    {
    	sprintf(msg, "Msg value: %d\r\n", (uint16_t)event.value.v);
    	HAL_UART_Transmit(&huart1, (uint8_t*)msg, strlen(msg), HAL_MAX_DELAY);
    }
  }
  /* USER CODE END MsgConsumerTask */
}
```



## Mail Queues

Mail Queues is similar to message queues in FreeRTOS. Both are FIFO queues and thread safe. But message queues stores the value, while mail queues stores pointer. To manage the memory storing mails, memory Pool is used to allocate and release the memory block of mails. Although API about mail queues has been integrated into CMSIS-RTOS, STM32Cube doesn’t provide the graphical configuration of mail queues. We need to create it by our own.

### API

The definition, creation, enqueues and dequeues of mail queues is similar to message queue:

```c
osMessageQId osMessageCreate (const osMessageQDef_t *queue_def, osThreadId thread_id)
osStatus osMessagePut (osMessageQId queue_id, uint32_t info, uint32_t millisec)
osEvent osMessageGet (osMessageQId queue_id, uint32_t millisec)
```

Even the macro in definition and creation is similar, so the code can be:

```c
/* USER CODE BEGIN Variables */
typedef struct
{
  uint16_t var;
} mailStruct;
osMailQId mail01Handle;
/* USER CODE END Variables */
// ...
void MX_FREERTOS_Init(void) {
  // ...
  /* USER CODE BEGIN RTOS_QUEUES */
  /* add queues, ... */
  osMailQDef(mail01, 16, mailStruct);
  mail01Handle = osMailCreate(osMailQ(mail01), NULL);
  /* USER CODE END RTOS_QUEUES */
  // ...
}
```

We use memory pool to allocate and free the memories use to store mail:

```c
/* USER CODE BEGIN Header_MsgProducerTask */
/**
  * @brief  Function implementing the MsgProducer thread.
  * @param  argument: Not used 
  * @retval None
  */
/* USER CODE END Header_MsgProducerTask */
void MsgProducerTask(void const * argument)
{
  /* USER CODE BEGIN MsgProducerTask */
  mailStruct * mail;
  /* Infinite loop */
  for(;;)
  {
    osDelay(1000);
    mail = (mailStruct *)osMailAlloc(mail01Handle, osWaitForever);
    mail->var = 1;
    osMailPut(mail01Handle, mail);
    osDelay(2000);
    mail = (mailStruct *)osMailAlloc(mail01Handle, osWaitForever);
    mail->var = 2;
    osMailPut(mail01Handle, mail);
  }
  /* USER CODE END MsgProducerTask */
}

/* USER CODE BEGIN Header_MsgConsumerTask */
/**
* @brief Function implementing the MsgConsumer thread.
* @param argument: Not used
* @retval None
*/
/* USER CODE END Header_MsgConsumerTask */
void MsgConsumerTask(void const * argument)
{
  /* USER CODE BEGIN MsgConsumerTask */
  osEvent event;
  mailStruct * pMail;
  char msg[20];
  /* Infinite loop */
  for(;;)
  {
    event = osMailGet(mail01Handle, osWaitForever);
    if (event.status == osEventMail)
    {
      pMail = event.value.p;
      sprintf(msg, "Mail value: %d\r\n", pMail->var);
      HAL_UART_Transmit(&huart1, (uint8_t*)msg, strlen(msg), HAL_MAX_DELAY);
      osMailFree(mail01Handle, pMail);
    }
  }
  /* USER CODE END MsgConsumerTask */
}
```



## Memory Pools

Memory pools are fixed-size blocks of memory that are thread-safe. Actually, calling ``osMailAlloc`` and ``osMailFree`` means using memory pools to allocate and release memory. Usage of memory pools is simple and you can check https://www.keil.com/pack/doc/CMSIS/RTOS/html/group__CMSIS__RTOS__PoolMgmt.html for more example.



## Assignments

1. Finish the practice of the last lab: Using counting semaphore to solve the producer-consumer problem
2. Using mail queues to solve the producer-consumer problem
3. The buffer size of the producer-consumer problem is 4
4. Due on **December 4, 2019**