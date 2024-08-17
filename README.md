# SoftwareTricks
### Некоторые программные трюки
В данном документе распологаются некоторые программные трюки, позволяющие упростить программы или увеличить их скорость.  
Все примеры написаны на языке ObjectPascal, но не используют каких-либо специфичных фич.  

### Sign
#### Определение знака числа без использования условных преходов(ветвлений)
**Идея: Ветвления это дорого(медленно)**  

Наивная реализация:
```Pascal
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
Версия основанная на трюке в виде того, что при приведении Boolean к Integer мы получим 1 для True и 0 для False:
```Pascal
function Sign(X: Integer): Integer;
begin
  Result := Integer(X > 0) - Integer(X < 0);
end;
```
Сравнение дизассемблера (FPC, O2):
```Pascal
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
```Pascal
function Compare(A, B: Integer): Integer;
begin
  Result := Integer(A > B) - Integer(A < B);
end;
```

### SAR
#### Знаковый сдвиг вправо
**Идея: в большинстве языков операторы ```>>(shr)``` и ```<<(shl)``` определены только для беззнаковых чисел, однако, при портировании "оптимизированного" кода с языков C/C++, часто необходим знаковый/арифметический сдвиг ```>>(sar)``` с расширением знака.**  

Интересно то, что в C++ это неспецифицированно, и такой код не обязан работать, но любителям оптимизации, с заменой ```/(div)``` на ```>>(sаr)```, при делении на степерь двойки, это неведомо. (https://learn.microsoft.com/en-us/cpp/cpp/left-shift-and-right-shift-operators-input-and-output).  
```
The result of a right-shift of a signed negative number is implementation-dependent.  
Although the Microsoft C++ compiler uses the sign bit to fill vacated bit positions,  
there is no guarantee that other implementations also do so.
```
Более того, для отрицательных чисел ```>>``` и ```/``` НЕ ЭКВИВАЛЕНТНЫ:
```C
#include <stdio.h>

int main() 
{
    printf("%i\n", -10 >> 2);
    printf("%i\n", -10 / 4);
    return 0;
}
>> -3
>> -2
```
Можете передать привет, советчикам по "оптимизации", с их крутыми советами из 70-х.  

Тем не менее, иногда надо портировать "оптимизированный" код 1 в 1, с сохранением этого поведения, ниже представленны несколько эквивалентов ```>>(sar)``` с расширемем знака для 32-х битных чисел, для сред и языков, где сдвиг - всегда беззнаковый.
```Pascal
function Sar(Number: Integer; Shift: Integer): Integer; inline;
var
  Mask: Integer;
begin
  if Shift = 0 then
    Exit(Number);// ничего не делаем при нулевом сдвиге

  Mask := -((Number shr (SizeOf(Integer) * 8 - 1)) and 1);
  Result := (Number shr Shift) or (Mask shl (SizeOf(Integer) * 8 - Shift));
end;
```
```Pascal
function Sar(Number: Integer; Shift: Integer): Integer; inline;
var
  SignMask, ShiftMask: Integer;
begin
  // Вычисляем маску знака числа: -1 для отрицательных, 0 для неотрицательных
  SignMask := -((Number shr (SizeOf(Integer) * 8 - 1)) and 1);
  // Вычисляем маску сдвига: 0 для нулевого сдвига, -1 для ненулевого
  ShiftMask := -Ord(Shift <> 0);
  // Применяем маски к результату сдвига
  Result := (Number shr Shift) or ((SignMask and ShiftMask) shl (SizeOf(Integer) * 8 - Shift));
end;
```
```Pascal
function Sar(Number: Integer; Shift: Integer): Integer; inline;
begin
  if Shift = 0 then
    Result := Number
  else
  begin
    if Number < 0 then
      Result := (Number shr Shift) or ((not 0) shl (SizeOf(Integer) * 8 - Shift))
    else
      Result := Number shr Shift;
  end;
end;
```
```Pascal
function Sar(Number: Integer; Shift: Integer): Integer; inline;
const
  SignMasks: array[0..1] of Cardinal = (0, $FFFFFFFF);
  ShiftMasks: array[0..31] of Cardinal = (
    $00000000, $80000000, $C0000000, $E0000000, $F0000000, $F8000000, $FC000000, $FE000000,
    $FF000000, $FF800000, $FFC00000, $FFE00000, $FFF00000, $FFF80000, $FFFC0000, $FFFE0000,
    $FFFF0000, $FFFF8000, $FFFFC000, $FFFFE000, $FFFFF000, $FFFFF800, $FFFFFC00, $FFFFFE00,
    $FFFFFF00, $FFFFFF80, $FFFFFFC0, $FFFFFFE0, $FFFFFFF0, $FFFFFFF8, $FFFFFFFC, $FFFFFFFE
  );
var
  SignMask, ShiftMask: Integer;
begin
  // Вычисляем маску знака числа: -1 для отрицательных, 0 для неотрицательных
  SignMask := SignMasks[Number shr (SizeOf(Integer) * 8 - 1) and 1];
  // Вычисляем маску сдвига из таблицы
  ShiftMask := ShiftMasks[Shift];
  // Применяем маски к результату сдвига
  Result := (Number shr Shift) or (SignMask and ShiftMask);
end;
```
Ну и конечно-же, никто не мешает вам, просто вызывать ассемблерный код с операцией ```SAR```(http://www.club155.ru/x86cmd/SAR):
```Pascal
function Sar(Number: Integer; Shift: Integer): Integer;
asm
  {$IFDEF CPUX64}
  mov eax,ecx
  mov ecx,edx
  sar eax,cl
  {$ELSE}
  mov ecx,edx
  sar eax,cl
  {$ENDIF}
end;
```
Отображение разницы:
```Pascal
program Test;

{$APPTYPE CONSOLE}
{$R *.res}

function Sar(Number: Integer; Shift: Integer): Integer; inline;
begin
  if Shift = 0 then
    Result := Number
  else
  begin
    if Number < 0 then
      Result := (Number shr Shift) or ((not 0) shl (SizeOf(Integer) * 8 - Shift))
    else
      Result := Number shr Shift;
  end;
end;

var
  I: Integer;
begin
  Writeln('sar | div');
  Writeln('---------');
  for I := -10 to 10 do
  begin
    Writeln(Sar(I, 1):3, ' |', I div 2:3);
  end;
  Readln;
end.
```
```
sar | div
---------
 -5 | -5
 -5 | -4
 -4 | -4
 -4 | -3
 -3 | -3
 -3 | -2
 -2 | -2
 -2 | -1
 -1 | -1
 -1 |  0 <--- \(O_o)/
  0 |  0
  0 |  0
  1 |  1
  1 |  1
  2 |  2
  2 |  2
  3 |  3
  3 |  3
  4 |  4
  4 |  4
  5 |  5
```
Вывод: 
1) Критически относитесь к советам об оптимизации, особенно к древним "трюкам" времен PDP-11.  
2) Перед "оптимизацией" деления знаковых чисел, убедитесь что вам точно подойдет ```sar``` вместо настоящего деления.
3) Не страдайте оптимизацией.  

