# 常用代码范式

> 嵌入式 C 开发中反复出现的经典编程模式，掌握这些范式可大幅提升代码质量。

## 📌 范式清单

1. 状态机（FSM）
2. 环形缓冲区（Ring Buffer）
3. 观察者 / 回调注册
4. 任务调度器（超轻量）
5. 参数校验防御式编程

---

## 🔑 状态机（FSM）

充电桩固件的核心架构，用于管理充电流程状态。

```c
/* 状态枚举 */
typedef enum {
    STATE_IDLE = 0,     /* 空闲等待插枪 */
    STATE_CONNECTED,    /* 已插枪，等待启动 */
    STATE_CHARGING,     /* 充电中 */
    STATE_FAULT,        /* 故障 */
} ChargerState;

/* 状态转移 */
void fsm_update(ChargerState *state, uint8_t event)
{
    switch (*state) {
        case STATE_IDLE:
            if (event == EVENT_GUN_IN) *state = STATE_CONNECTED;
            break;
        case STATE_CONNECTED:
            if (event == EVENT_START) *state = STATE_CHARGING;
            break;
        /* ... */
        default:
            break;
    }
}
```

---

## 🔑 环形缓冲区（Ring Buffer）

用于 UART 接收数据的无锁缓冲。

```c
#define RING_BUF_SIZE 256

typedef struct {
    uint8_t  buf[RING_BUF_SIZE];
    uint16_t head;   /* 写指针 */
    uint16_t tail;   /* 读指针 */
} RingBuffer;

/* 写入一字节 */
void ring_push(RingBuffer *rb, uint8_t data)
{
    rb->buf[rb->head] = data;
    rb->head = (rb->head + 1) % RING_BUF_SIZE;
}

/* 读取一字节 */
int ring_pop(RingBuffer *rb, uint8_t *data)
{
    if (rb->head == rb->tail) return -1;  /* 空 */
    *data = rb->buf[rb->tail];
    rb->tail = (rb->tail + 1) % RING_BUF_SIZE;
    return 0;
}
```

---

## 📝 待补充内容

- 观察者模式实现
- 超轻量任务调度器（无 RTOS 场景）
- 防御式编程与断言（assert）
- 单例模式（驱动实例管理）
