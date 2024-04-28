#channel #channels #go #Goroutine

Каналы созданы для общения горутин, поэтому аллоцировать память для них, нужно в heap. 

Синтаксис для создания канала:
```go
ch := make(chan type, size)
```

Структура канала: 
![[Снимок экрана 2024-04-28 в 18.20.13.png]]
- qcount - кол-во элементов , которые хранятся в буфере
- dataqsize - размерность буфера
- buf - ссылка на буфер 
- closed - закрыт ли канал (uint32 из-за атомиков)
- elemsize - размер элементов
- elemtype - тип элемента
- recvq - указатель на связанный список
- sendq -  указатель на связанный список
- recx - номер ячейки буфера
- sendx - номер ячейки буфера
- lock - mutex

```go
type hchan struct {

	qcount   uint // total data in the queue
	dataqsiz uint // size of the circular queue
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	elemsize uint16
	closed   uint32
	elemtype *_type // element type
	sendx    uint   // send index
	recvx    uint   // receive index
	recvq    waitq  // list of recv waiters
	sendq    waitq  // list of send waiters
	lock mutex
}
```

Как сделать семантику FIFO?
- создадим кольцевую очередь.

Важно, данные в канал копируются, кроме ссылочного формата.

## Что делать при переполнении буфера?
---
Гугл решил эту проблему через парковку рутин. При достижении лимита, выполняется функция `gopark()`.
Она обращается к планировщику, который меняет состояние рутины с running на waiting. После разрывается связь с OS Thread. Если в runq есть другая рутина, то тред переключается на нее. 

### А как продолжить выполнение рутины?
Опять-таки отправляем ее в sendq. Сам sendq представляет собой ссылку на waitq в который ссылка на начало и конец. Начало и конец представляет собой sudog. 
![[Снимок экрана 2024-04-28 в 18.42.36.png]]
G - горутина. 
elem - данные, которые надо передать.

Ну и sudog передаем в sendq, чтоб ее могли прочитать.

Заметим, что recv(reader) занимается запуском горутины. 

Как он это делает?
Он выполняет `goready()`, функция полная противоположность `gopark()`, после ставит статус runnable и кладет рутину в runq. ![[Снимок экрана 2024-04-28 в 18.48.15.png]]

## Unbuffered channels
Данные отправляются напрямую , через обмен контекстов горутин, функция `sendDirect()`
## Sum up 
- ﻿﻿goroutine-safe: hchan mutex.
- ﻿﻿хранение элементов, семантика FIFO: hchan buf.
- ﻿﻿передача данных между горутинами: sendDirect, operations with buf.
- ﻿﻿блокировка горутин senda / recva, sudog.
- calls to scheduler: gopark(), goready. 
- Запись или чтение из пустого канала блокирует горутину навсегда. 
