---
title: "git merge conflict"
date: 2021-09-15T01:16:41+03:00
draft: false
---

git режет изменения на блоки и если два блока соседствуют, то будет конфликт, если же блоки разделяет общая линия, то конфликта не будет и слияние будет автоматическим

### пример где есть неизменная линия

master|test|description
---|---|---
1||изначально в файле только 1
1<br>2||добавили 2
1<br>2<br>3||добавили 3
11<br>2<br>3|1<br>2<br>33|в master поменяли первую строчку, в test поменяли последнюю
11<br>2<br>33||git merge test<br>слилось автомически<br>т.к. 2 общая и разделяет блоки изменений

### тоже самое, только блоки соседствуют

master|test|description
---|---|---
1||изначально в файле только 1
1<br>2||добавили 2
11<br>2|1<br>22|в master поменяли первую строчку, в test поменяли последнюю
11<br>22||git merge test<br>будет конфликт<br>т.к. блоки изменений соседствуют