---
title: "Golang Naming"
date: 2021-08-07T10:17:07+03:00
draft: false
---
```
for _, tt := range tests {
}
```
Смысл в удваивании имени переменной когда итерируем по слайсу.
Это применяется в тестах.

```
tests := []struct {
	give     string
	wantHost string
	wantPort string
}{
	{
		give:     "",
		wantHost: "",
		wantPort: "",
	},
}

for _, tt := range tests {
  t.Run(tt.give, func(t *testing.T) {
    host, port, err := net.SplitHostPort(tt.give)
    require.NoError(t, err)
    assert.Equal(t, tt.wantHost, host)
    assert.Equal(t, tt.wantPort, port)
  })
}
```
