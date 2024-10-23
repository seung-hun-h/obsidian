- 오라클은 빈 문자열(길이가 0)과 NULL을 동일하게 본다

```sql
SELECT ''
FROM DUAL;
```
```result
NULL
```
- 길이가 0이 아닌 공백 문자열은 공백 문자열 그대로 본다

```sql
SELECT '    '
FROM DUAL;
```
```result
'    '
```

