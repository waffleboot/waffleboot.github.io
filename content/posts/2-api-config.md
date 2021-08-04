---
title: "Про параметры функции, публикуемых в открытый доступ"
date: 2021-08-05T00:40:05+03:00
draft: false
---

Изучал код worker пула [https://github.com/ahmetask/worker](https://github.com/ahmetask/worker) и там есть такая функция NewWorkerPool(maxWorkers, jobQueueCapacity):

```
func NewWorkerPool(maxWorkers int, jobQueueCapacity int) *Pool
```

Я предложил сделать отдельную структуру под конфигурацию workerPoolConfig и worker пул иницилизировать не через параметры, а через функциональные опции, через ...Opts. Добавить функцию buildWorkerPoolConfig, которая бы эти ...Opts читала и создавала конфиг, дополнительно валидируя параметры. Заслал PR, чел отказался фиксить по причине, что сломается обратная совместимость, а либа и так простая, так что нет смысла.

```
type config struct
type opts func(* config)
func WithMaxWorkers(maxWorkers int) opts
func WithJobQueueCapacity(jobQueueCapacity int) opts
func buildConfig(opts ...opts) * config {
	cfg := &config{}
	for _, opt := range opts {
		opt(cfg)
	}
	// валидация cfg
	return cfg
}
func NewWorkerPool(opts ...opts) *Pool {
	cfg := buildConfig(opts...)
}

pool := worker.NewWorkerPool(
  worker.WithMaxWorkers(4),
  worker.WithJobQueueCapacity(4))
```

Вот и возникает вопрос. Если бы надо было добавить дополнительный параметр, то все равно пришлось бы править эту функцию или создавать еще одну, которая воспринимала бы дополнительный параметр. А с функциональными опциями такой проблемы нет. Старый код как работал, так и будет работать, а для нового кода просто еще одна опция добавляется, с которой мы умеем работать.
