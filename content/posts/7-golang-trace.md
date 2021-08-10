---
title: "Golang Trace"
date: 2021-08-10T12:15:17+03:00
draft: false
---

это программа

```
func main() {
	f, err := os.Create("trace.out")
	if err != nil {
		log.Fatal(err)
	}
	defer f.Close()
	if err := trace.Start(f); err != nil {
		log.Fatal(err)
	}
	go func() {
		// runtime.PreemptNS(1000)
		for {
			_x++
		}
	}()
	<-time.After(3 * time.Second)
	runtime.GC()
	defer trace.Stop()
}

```

это содержимое trace.out при запуске с GOMAXPROCS=1

```
  // сначала идет список горутин на момент запуска trace.Start
  // G1 в статусе running, остальные в статусе waiting потому что P у нас только один
  1: GoCreate       /Users/yangand/Workspace/go_infinite_loop/main.go:19 g=1 stack=2 linked GoStart@11 
  2: GoCreate       /Users/yangand/Workspace/go_infinite_loop/main.go:19 g=2 stack=3 
  3: GoWaiting   G2 g=2 
  4: GoCreate       /Users/yangand/Workspace/go_infinite_loop/main.go:19 g=3 stack=4 
  5: GoWaiting   G3 g=3 linked GoUnblock@312 
  6: GoCreate       /Users/yangand/Workspace/go_infinite_loop/main.go:19 g=4 stack=5 
  7: GoWaiting   G4 g=4 linked GoUnblock@299 
  8: GoCreate       /Users/yangand/Workspace/go_infinite_loop/main.go:19 g=5 stack=6 
  9: GoWaiting   G5 g=5 

 // это пишет tracer и просто отбивка на каком P сидит текущая горутина и что за горутина
 // + показывается gomaxprocs
 10: ProcStart      thread=0 reason=1 
 11: GoStart     G1 /usr/local/Cellar/go/1.16.6/libexec/src/runtime/proc.go:115 g=1 seq=0 linked GoBlockRecv@15 
 12: Gomaxprocs  G1 /Users/yangand/Workspace/go_infinite_loop/main.go:19 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/proc.go:1003 procs=1 
 
 // main горутина создает две горутины
 // G6 создает tracer и эта горутина пишет trace.out
 // G7 наш бесконечный цикл

 13: GoCreate    G1 /Users/yangand/Workspace/go_infinite_loop/main.go:19 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/trace/trace.go:127 g=6 stack=8 linked GoStart@18 
 14: GoCreate    G1 /Users/yangand/Workspace/go_infinite_loop/main.go:22 g=7 stack=10 linked GoStart@16 

 // наша main горутина уперлась в чтение канала с таймера
 15: GoBlockRecv G1 /Users/yangand/Workspace/go_infinite_loop/main.go:28 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/chan.go:439 linked GoUnblock@284 

 // пошел работать бесконечный цикл и спустя вроде 10ms он будет вытеснен
 // работает sysmon поток/горутина, она смотрит сколько каждый P работает и если больше 10ms занимается
 // одной горутиной, то шлет M-потоку этой горутины SIGUSR сигнал
 // OS останавливает этот поток, обработчик прерывания делает schedule и G7 вытесняется
 16: GoStart     G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 g=7 seq=0 linked GoPreempt@17 
 17: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@23 

 // G6 получает возможность работать
 // она делает syscall, запись в файл
 // при этом лочится на потоке, на P, при вызове syscall горутина пишет что уходит в syscall и лочится
 // sysmon опять же крутится и смотрит за всеми P которые долго выполняются или в syscall
 // видит что P подвис и отрывает P от этого потока, потому что надо выполнять горутины
 // зависшие в этом P
 18: GoStart     G6 /usr/local/Cellar/go/1.16.6/libexec/src/runtime/trace/trace.go:127 g=6 seq=0 linked GoSysBlock@20 
 19: GoSysCall   G6 /usr/local/Cellar/go/1.16.6/libexec/src/runtime/trace/trace.go:133 -> /usr/local/Cellar/go/1.16.6/libexec/src/syscall/zsyscall_darwin_amd64.go:1734 linked GoSysExit@34 
 20: GoSysBlock  G6 

 // это собственно отрыв P от M и создание (или выборка из кэша потоков) новой M
 // раньше P сидел на thread=0, сейчас же P назначается новый поток thread=2
 21: ProcStop       
 22: ProcStart      thread=2 reason=0 

 // P может работать и у нас в очереди P сидит G7 и она ставится на выполнение
 // каждый раз она будет вытесняться и ставится на выполнение опять потому что больше нет горутин
 23: GoStart     G7 g=7 seq=0 linked GoPreempt@24 
 24: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@25 
 25: GoStart     G7 g=7 seq=0 linked GoPreempt@26 
 26: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@27 
 27: GoStart     G7 g=7 seq=0 linked GoPreempt@28 
 28: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@29 
 29: GoStart     G7 g=7 seq=0 linked GoPreempt@30 
 30: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@31 
 31: GoStart     G7 g=7 seq=0 linked GoPreempt@32 
 32: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@33 
 33: GoStart     G7 g=7 seq=0 linked GoPreempt@35 

 // тут G6 выходит из syscall
 34: GoSysExit   G6 P1000003 g=6 seq=2 ts=195024786635 linked GoStart@36 

 // G7 опять вытесняется
 // и вместо нее начнется выполняться G6
 35: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@38 

 // G6 записала часть данных
 // но буфер новыми данными еще не заполнился, поэтому чтобы не крутится в цикле
 // G6 паркуется до полного буфера
 36: GoStart     G6 g=6 seq=0 linked GoBlock@37 
 37: GoBlock     G6 /usr/local/Cellar/go/1.16.6/libexec/src/runtime/trace/trace.go:129 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/trace.go:411 

 // G7 опять начинает работать и вытесняться
 38: GoStart     G7 g=7 seq=0 linked GoPreempt@39 
 39: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@40 
 40: GoStart     G7 g=7 seq=0 linked GoPreempt@41 
 41: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@42 
 42: GoStart     G7 g=7 seq=0 linked GoPreempt@43 
 43: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@44 
 44: GoStart     G7 g=7 seq=0 linked GoPreempt@45 
 45: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@46 
 46: GoStart     G7 g=7 seq=0 linked GoPreempt@47 
 47: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@48 
 48: GoStart     G7 g=7 seq=0 linked GoPreempt@49 
 49: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@50 
 50: GoStart     G7 g=7 seq=0 linked GoPreempt@51 
 51: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@52 
 52: GoStart     G7 g=7 seq=0 linked GoPreempt@53 
 53: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@54 
 54: GoStart     G7 g=7 seq=0 linked GoPreempt@55 
 55: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@56 
 56: GoStart     G7 g=7 seq=0 linked GoPreempt@57 
 57: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@58 
 58: GoStart     G7 g=7 seq=0 linked GoPreempt@59 
 59: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@60 
 60: GoStart     G7 g=7 seq=0 linked GoPreempt@61 
 61: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@62 
 62: GoStart     G7 g=7 seq=0 linked GoPreempt@63 
 63: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@64 
 64: GoStart     G7 g=7 seq=0 linked GoPreempt@65 
 65: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@66 
 66: GoStart     G7 g=7 seq=0 linked GoPreempt@67 
 67: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@68 
 68: GoStart     G7 g=7 seq=0 linked GoPreempt@69 
 69: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@70 
 70: GoStart     G7 g=7 seq=0 linked GoPreempt@71 
 71: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@72 
 72: GoStart     G7 g=7 seq=0 linked GoPreempt@73 
 73: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@74 
 74: GoStart     G7 g=7 seq=0 linked GoPreempt@75 
 75: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@76 
 76: GoStart     G7 g=7 seq=0 linked GoPreempt@77 
 77: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@78 
 78: GoStart     G7 g=7 seq=0 linked GoPreempt@79 
 79: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@80 
 80: GoStart     G7 g=7 seq=0 linked GoPreempt@81 
 81: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@82 
 82: GoStart     G7 g=7 seq=0 linked GoPreempt@83 
 83: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@84 
 84: GoStart     G7 g=7 seq=0 linked GoPreempt@85 
 85: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@86 
 86: GoStart     G7 g=7 seq=0 linked GoPreempt@87 
 87: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@88 
 88: GoStart     G7 g=7 seq=0 linked GoPreempt@89 
 89: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@90 
 90: GoStart     G7 g=7 seq=0 linked GoPreempt@91 
 91: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@92 
 92: GoStart     G7 g=7 seq=0 linked GoPreempt@93 
 93: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@94 
 94: GoStart     G7 g=7 seq=0 linked GoPreempt@95 
 95: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@96 
 96: GoStart     G7 g=7 seq=0 linked GoPreempt@97 
 97: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@98 
 98: GoStart     G7 g=7 seq=0 linked GoPreempt@99 
 99: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@100 
100: GoStart     G7 g=7 seq=0 linked GoPreempt@101 
101: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@102 
102: GoStart     G7 g=7 seq=0 linked GoPreempt@103 
103: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@104 
104: GoStart     G7 g=7 seq=0 linked GoPreempt@105 
105: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@106 
106: GoStart     G7 g=7 seq=0 linked GoPreempt@107 
107: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@108 
108: GoStart     G7 g=7 seq=0 linked GoPreempt@109 
109: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@110 
110: GoStart     G7 g=7 seq=0 linked GoPreempt@111 
111: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@112 
112: GoStart     G7 g=7 seq=0 linked GoPreempt@113 
113: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@114 
114: GoStart     G7 g=7 seq=0 linked GoPreempt@115 
115: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@116 
116: GoStart     G7 g=7 seq=0 linked GoPreempt@117 
117: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@118 
118: GoStart     G7 g=7 seq=0 linked GoPreempt@119 
119: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@120 
120: GoStart     G7 g=7 seq=0 linked GoPreempt@121 
121: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@122 
122: GoStart     G7 g=7 seq=0 linked GoPreempt@123 
123: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@124 
124: GoStart     G7 g=7 seq=0 linked GoPreempt@125 
125: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@126 
126: GoStart     G7 g=7 seq=0 linked GoPreempt@127 
127: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@128 
128: GoStart     G7 g=7 seq=0 linked GoPreempt@129 
129: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@130 
130: GoStart     G7 g=7 seq=0 linked GoPreempt@131 
131: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@132 
132: GoStart     G7 g=7 seq=0 linked GoPreempt@133 
133: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@134 
134: GoStart     G7 g=7 seq=0 linked GoPreempt@135 
135: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@136 
136: GoStart     G7 g=7 seq=0 linked GoPreempt@137 
137: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@138 
138: GoStart     G7 g=7 seq=0 linked GoPreempt@139 
139: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@140 
140: GoStart     G7 g=7 seq=0 linked GoPreempt@141 
141: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@142 
142: GoStart     G7 g=7 seq=0 linked GoPreempt@143 
143: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@144 
144: GoStart     G7 g=7 seq=0 linked GoPreempt@145 
145: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@146 
146: GoStart     G7 g=7 seq=0 linked GoPreempt@147 
147: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@148 
148: GoStart     G7 g=7 seq=0 linked GoPreempt@149 
149: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@150 
150: GoStart     G7 g=7 seq=0 linked GoPreempt@151 
151: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@152 
152: GoStart     G7 g=7 seq=0 linked GoPreempt@153 
153: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@154 
154: GoStart     G7 g=7 seq=0 linked GoPreempt@155 
155: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@156 
156: GoStart     G7 g=7 seq=0 linked GoPreempt@157 
157: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@158 
158: GoStart     G7 g=7 seq=0 linked GoPreempt@159 
159: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@160 
160: GoStart     G7 g=7 seq=0 linked GoPreempt@161 
161: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@162 
162: GoStart     G7 g=7 seq=0 linked GoPreempt@163 
163: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@164 
164: GoStart     G7 g=7 seq=0 linked GoPreempt@165 
165: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@166 
166: GoStart     G7 g=7 seq=0 linked GoPreempt@167 
167: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@168 
168: GoStart     G7 g=7 seq=0 linked GoPreempt@169 
169: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@170 
170: GoStart     G7 g=7 seq=0 linked GoPreempt@171 
171: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@172 
172: GoStart     G7 g=7 seq=0 linked GoPreempt@173 
173: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@174 
174: GoStart     G7 g=7 seq=0 linked GoPreempt@175 
175: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@176 
176: GoStart     G7 g=7 seq=0 linked GoPreempt@177 
177: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@178 
178: GoStart     G7 g=7 seq=0 linked GoPreempt@179 
179: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@180 
180: GoStart     G7 g=7 seq=0 linked GoPreempt@181 
181: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@182 
182: GoStart     G7 g=7 seq=0 linked GoPreempt@183 
183: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@184 
184: GoStart     G7 g=7 seq=0 linked GoPreempt@185 
185: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@186 
186: GoStart     G7 g=7 seq=0 linked GoPreempt@187 
187: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@188 
188: GoStart     G7 g=7 seq=0 linked GoPreempt@189 
189: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@190 
190: GoStart     G7 g=7 seq=0 linked GoPreempt@191 
191: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@192 
192: GoStart     G7 g=7 seq=0 linked GoPreempt@193 
193: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@194 
194: GoStart     G7 g=7 seq=0 linked GoPreempt@195 
195: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@196 
196: GoStart     G7 g=7 seq=0 linked GoPreempt@197 
197: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@198 
198: GoStart     G7 g=7 seq=0 linked GoPreempt@199 
199: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@200 
200: GoStart     G7 g=7 seq=0 linked GoPreempt@201 
201: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@202 
202: GoStart     G7 g=7 seq=0 linked GoPreempt@203 
203: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@204 
204: GoStart     G7 g=7 seq=0 linked GoPreempt@205 
205: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@206 
206: GoStart     G7 g=7 seq=0 linked GoPreempt@207 
207: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@208 
208: GoStart     G7 g=7 seq=0 linked GoPreempt@209 
209: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@210 
210: GoStart     G7 g=7 seq=0 linked GoPreempt@211 
211: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@212 
212: GoStart     G7 g=7 seq=0 linked GoPreempt@213 
213: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@214 
214: GoStart     G7 g=7 seq=0 linked GoPreempt@215 
215: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@216 
216: GoStart     G7 g=7 seq=0 linked GoPreempt@217 
217: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@218 
218: GoStart     G7 g=7 seq=0 linked GoPreempt@219 
219: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@220 
220: GoStart     G7 g=7 seq=0 linked GoPreempt@221 
221: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@222 
222: GoStart     G7 g=7 seq=0 linked GoPreempt@223 
223: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@224 
224: GoStart     G7 g=7 seq=0 linked GoPreempt@225 
225: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@226 
226: GoStart     G7 g=7 seq=0 linked GoPreempt@227 
227: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@228 
228: GoStart     G7 g=7 seq=0 linked GoPreempt@229 
229: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@230 
230: GoStart     G7 g=7 seq=0 linked GoPreempt@231 
231: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@232 
232: GoStart     G7 g=7 seq=0 linked GoPreempt@233 
233: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@234 
234: GoStart     G7 g=7 seq=0 linked GoPreempt@235 
235: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@236 
236: GoStart     G7 g=7 seq=0 linked GoPreempt@237 
237: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@238 
238: GoStart     G7 g=7 seq=0 linked GoPreempt@239 
239: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@240 
240: GoStart     G7 g=7 seq=0 linked GoPreempt@241 
241: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@242 
242: GoStart     G7 g=7 seq=0 linked GoPreempt@243 
243: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@244 
244: GoStart     G7 g=7 seq=0 linked GoPreempt@245 
245: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@246 
246: GoStart     G7 g=7 seq=0 linked GoPreempt@247 
247: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@248 
248: GoStart     G7 g=7 seq=0 linked GoPreempt@249 
249: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@250 
250: GoStart     G7 g=7 seq=0 linked GoPreempt@251 
251: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@252 
252: GoStart     G7 g=7 seq=0 linked GoPreempt@253 
253: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@254 
254: GoStart     G7 g=7 seq=0 linked GoPreempt@255 
255: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@256 
256: GoStart     G7 g=7 seq=0 linked GoPreempt@257 
257: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@258 
258: GoStart     G7 g=7 seq=0 linked GoPreempt@259 
259: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@260 
260: GoStart     G7 g=7 seq=0 linked GoPreempt@261 
261: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@262 
262: GoStart     G7 g=7 seq=0 linked GoPreempt@263 
263: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@264 
264: GoStart     G7 g=7 seq=0 linked GoPreempt@265 
265: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@266 
266: GoStart     G7 g=7 seq=0 linked GoPreempt@267 
267: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@268 
268: GoStart     G7 g=7 seq=0 linked GoPreempt@269 
269: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@270 
270: GoStart     G7 g=7 seq=0 linked GoPreempt@271 
271: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@272 
272: GoStart     G7 g=7 seq=0 linked GoPreempt@273 
273: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@274 
274: GoStart     G7 g=7 seq=0 linked GoPreempt@275 
275: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@276 
276: GoStart     G7 g=7 seq=0 linked GoPreempt@277 
277: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@278 
278: GoStart     G7 g=7 seq=0 linked GoPreempt@279 
279: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@280 
280: GoStart     G7 g=7 seq=0 linked GoPreempt@281 
281: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@282 
282: GoStart     G7 g=7 seq=0 linked GoPreempt@283 
283: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@294 

// G7 снова вытеснили
// при этом sysmon проверяет в ходе своей работы таймеры
// и видит что наш таймер сработал, дергается функция этого таймера sendtime
// которая пишет now() в канал и G1 стоящая в очереди на чтение у этого канала будится
// и ставится на выполнение
284: GoUnblock      g=1 seq=0 linked GoStart@285 

// G1 начинает работать после сработки таймера
285: GoStart     G1 g=1 seq=0 linked GoSysBlock@289 

// это мы пустили runtime.GC
286: GCStart     P1000004 /Users/yangand/Workspace/go_infinite_loop/main.go:29 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/mgc.go:1159 seq=0 linked GCDone@313 

// runtime.GC запускает gcmark потоки по числу gomaxprocs
// и дергает cgo yield, видимо для какой-то кооперации с CGO по части сборки мусора
// как и tracer тут пишется что горутина уходит в syscall и блокируется
287: GoCreate    G1 /Users/yangand/Workspace/go_infinite_loop/main.go:29 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/mgc.go:1835 g=8 stack=19 linked GoStart@292 
288: GoSysCall   G1 /Users/yangand/Workspace/go_infinite_loop/main.go:29 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/mgc.go:1837 linked GoSysExit@295 
289: GoSysBlock  G1 

// так как горутина ушла в syscall, то P опять подвис
// ищется M куда его подвесить и у нас есть свободный thread=0 который раньше висел в syscall у tracer
// мы просто берем его из thread pool
290: ProcStop       
291: ProcStart      thread=0 reason=0 

// тут у нас G8 горутина просыпается
// она ждет сигнала от CGO, поэтому паркуется
292: GoStart     G8 /usr/local/Cellar/go/1.16.6/libexec/src/runtime/mgc.go:1877 g=8 seq=0 linked GoBlock@293 
293: GoBlock     G8 linked GoUnblock@303 

// опять проснулся бесконечный цикл
// он потом будет вытеснен как обычно
294: GoStart     G7 g=7 seq=0 linked GoPreempt@296 
// G1 вернулась из CGO и будет поставлена на выполнение
295: GoSysExit   G1 P1000003 g=1 seq=4 ts=195114073646 linked GoStart@297 
296: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:25 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 linked GoStart@306 

297: GoStart     G1 g=1 seq=0 linked GoBlock@302 

// я так понимаю STW фаза с kindid=1
298: GCSTWStart     kindid=1 kind=sweep termination linked GCSTWDone@301 

// G1 разблокирует G4
299: GoUnblock   G1 g=4 seq=0 linked GoStart@321 
300: Gomaxprocs  G1 /Users/yangand/Workspace/go_infinite_loop/main.go:29 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/mgc.go:1442 procs=1 
301: GCSTWDone      
302: GoBlock     G1 /Users/yangand/Workspace/go_infinite_loop/main.go:29 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/proc.go:342 linked GoUnblock@315 
303: GoUnblock      g=8 seq=0 linked GoStartLabel@304 

304: GoStartLabelG8 g=8 seq=3 labelid=3 label=GC (fractional) linked GoBlock@305 
305: GoBlock     G8 linked GoUnblock@308 

// опять G7 квант времени получил
306: GoStart     G7 g=7 seq=0 linked GoPreempt@307 
307: GoPreempt   G7 /Users/yangand/Workspace/go_infinite_loop/main.go:24 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/preempt_amd64.s:51 

308: GoUnblock      g=8 seq=0 linked GoStartLabel@309 
309: GoStartLabelG8 g=8 seq=5 labelid=3 label=GC (fractional) linked GoBlock@318 

// это видимо тоже STW только kindid=0
310: GCSTWStart     kindid=0 kind=mark termination linked GCSTWDone@317 

// кто-то запросил память
311: HeapAlloc   G8 mem=37344 
312: GoUnblock   G8 g=3 seq=0 linked GoStart@319 
313: GCDone         

// это следующая цель по памяти, как-то связано с GOGC
314: HeapGoal    G8 mem=4194304 
315: GoUnblock   G8 g=1 seq=0 linked GoStart@323 
316: Gomaxprocs  G8 /usr/local/Cellar/go/1.16.6/libexec/src/runtime/mgc.go:2045 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/mgc.go:1746 procs=1 
317: GCSTWDone      
318: GoBlock     G8 

// не понимаю что за G3 и G4 горутины
319: GoStart     G3 /usr/local/Cellar/go/1.16.6/libexec/src/runtime/mgcsweep.go:156 g=3 seq=0 linked GoBlock@320 
320: GoBlock     G3 /usr/local/Cellar/go/1.16.6/libexec/src/runtime/mgcsweep.go:182 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/proc.go:342 

321: GoStart     G4 /usr/local/Cellar/go/1.16.6/libexec/src/runtime/mgcscavenge.go:252 g=4 seq=0 linked GoBlock@322 
322: GoBlock     G4 /usr/local/Cellar/go/1.16.6/libexec/src/runtime/mgcscavenge.go:314 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/proc.go:342 

// в конечном счете мы вернемся из runtime.GC обратно в main
// и trace.Stop финальным аккордом пишет GoSched отметку
323: GoStart     G1 g=1 seq=0 linked GoSched@324 
324: GoSched     G1 /Users/yangand/Workspace/go_infinite_loop/main.go:31 -> /usr/local/Cellar/go/1.16.6/libexec/src/runtime/trace.go:1084 

// а это времена выполнения горутин
// это main горутина, выполнялась мало, долго сидела под таймером, cgo syscall, gc
 1: Exec=208192 SchedWait=22935870 Block=3016748479 Syscall=34656 GC=48044603 Total=3065121465 runtime.main
 // не знаю что это
 2: SchedWait=3065115226 GC=48044603 Total=3065115226 
 
 // это какие-то горутины GC
 3: Exec=8992 SchedWait=141120 GC=48044603 Total=3065109626 runtime.bgsweep
 4: Exec=416 SchedWait=25344829 GC=48044603 Total=3065104474 runtime.bgscavenge
 5: SchedWait=3065099130 GC=48044603 Total=3065099130 
 
 // это горутина записи trace events в файл, видны syscall
 6: Exec=24534750 SchedWait=24649726 Syscall=121004052 GC=48044603 Total=3065041178 runtime/trace.Start.func1
 // бесконечный цикл
 7: Exec=3039109404 SchedWait=25917630 GC=48044603 Total=3065027034 main.main.func1
 // gcmark
 8: Exec=408416 SchedWait=26880 GC=48021435 Total=48183899 runtime.gcBgMarkWorker

```

