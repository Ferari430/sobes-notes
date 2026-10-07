Такие индексы применимы к составным типом, работа с которыми осуществляется с помощью ключей. Это массивы, jsonb и tsvector.

ТО есть типы составные в строках, и в запросе мы ищем наличие элементов в этих индексируемых строках.

==Такой индекс построен так же за счет дерева как и в b-tree но только хранит другие значения в листьях, адаптированные под текст.==

Это могут быть документы, которые содержат определенные слова. И из запроса мы вытаскиваем слово по которому будем искать.

## Когда использовать GIN:

```sql

-- 1. Полнотекстовый поиск
SELECT * FROM articles 
WHERE to_tsvector('english', body) @@ to_tsquery('quick & brown');

-- 2. Поиск в массивах
SELECT * FROM products 
WHERE tags @> ARRAY['electronics', 'sale'];

-- 3. JSON/JSONB
SELECT * FROM orders 
WHERE metadata @> '{"status": "shipped"}';

-- 4. Составные типы
SELECT * FROM users 
WHERE profile->'skills' ? 'PostgreSQL';
```


![Screenshot from 2025-12-15 19-58-09 1](../../../attachments/Screenshot%20from%202025-12-15%2019-58-09%201.png)