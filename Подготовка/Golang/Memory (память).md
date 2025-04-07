Полезные ссылки:
- [The Go Memory Model](https://go.dev/ref/mem)

---

1. Stack-и goroutine 
   [Stack managment](https://www.sobyte.net/post/2021-12/golang-stack-management/)
	1. С какого размера он начинается и как динамически он растёт?
		1. [https://habr.com/ru/companies/otus/articles/586108/](https://habr.com/ru/companies/otus/articles/586108/)
		2. [https://blog.cloudflare.com/how-stacks-are-handled-in-go/](https://blog.cloudflare.com/how-stacks-are-handled-in-go/)
	2. Что такое Contiguous stacks?
		1. [https://docs.google.com/document/u/0/d/1wAaf1rYoM4S4gtnPh0zOlGzWtrZFQ5suE8qr2sD8uWQ/mobilebasic?_immersive_translate_auto_translate=1](https://docs.google.com/document/u/0/d/1wAaf1rYoM4S4gtnPh0zOlGzWtrZFQ5suE8qr2sD8uWQ/mobilebasic?_immersive_translate_auto_translate=1)
	3. Что такое Segmented stack?
		1. [https://blog.cloudflare.com/how-stacks-are-handled-in-go/](https://blog.cloudflare.com/how-stacks-are-handled-in-go/)
		2. [https://golang-nuts.narkive.com/xfWAIezk/need-help-understanding-segmented-stack-and-stack-splitting](https://golang-nuts.narkive.com/xfWAIezk/need-help-understanding-segmented-stack-and-stack-splitting)
	4. Где Stack goroutine находится?
2. Аллокатор в Go
	1. Как он выделяет память?
3. Garbage collector (GC)
	1. Как он устроен?
	2. По какой модели он работает?
	3. Какие у него есть основные фазы?
	4. На каких фазах происходит полная остановка программы (stop the world)?
	5. Как можно управлять GC?
