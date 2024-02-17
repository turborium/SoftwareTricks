# SoftwareTricks
### Некоторые программные трюки
В данном документе распологаются некоторые программыне трюки, позволяющие упростить программы или увеличить их скорость.  
Все примеры написаны на языке ObjectPascal.  

### Sign
#### Определение знака числа без использования условных выражений
Наивная реализация:
```
function Sign(X: Integer): Integer;
begin
  if X = 0 then
    Result := 0
  else if X > 0 then
    Result := 1
  else
    Result := -1;
end;
```
Версия основанная на трюке в виде того, что при приведении Boolean к Integer мы получит 1 для True и 0 для False:
```
function Sign(X: Integer): Integer;
begin
  Result := Integer(X > 0) - Integer(X < 0);
end;
```
Сравнение дизассемблера (FPC, O2):
```
; Наивная реализация:
sign(longint):
        testl   %edi,%edi
        jne     .Lj6
        xorl    %eax,%eax
        ret
.Lj6:
        testl   %edi,%edi
        jng     .Lj9
        movl    $1,%eax
        ret
.Lj9:
        movl    $-1,%eax
        ret

; Версия с трюком:
sign(longint):
        testl   %edi,%edi
        setgb   %al
        andl    $255,%eax
        testl   %edi,%edi
        setlb   %dl
        andl    $255,%edx
        subl    %edx,%eax
        ret
```
Можно использовать и для сравнения чисел, например так:
```
function Compare(A, B: Integer): Integer;
begin
  Result := Integer(A > B) - Integer(A < B);
end;
```

