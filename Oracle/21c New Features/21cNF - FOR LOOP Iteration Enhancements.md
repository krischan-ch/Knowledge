## 21cNF - FOR LOOP Iteration Enhancements==

### Multiple Iterations

#### Multiple ranges in one for loop

Syntax

```SQL
begin
  for i in 1 .. 3, reverse 7 .. 9, 20 .. 22 loop
    dbms_output.put_line(i);
  end loop;
end;
/
```

Output

```SQL
1
2
3
9
8
7
20
21
22

PL/SQL procedure successfully completed.

SQL>
```

#### Specific step in iteration

We can now specify the value to increment by: 

Syntax

```SQL
begin
  for i in 1 .. 5 by 2, reverse 1 .. 5 by 2 loop
    dbms_output.put_line(i);
  end loop;
end;
/
```

Output

```SQL
1
3
5
5
3
1

PL/SQL procedure successfully completed.

SQL>
```

#### fractional stepped range iteration

Fractional iterators get rounded:

Syntax

```SQL
begin
  for i in 1.2 .. 2.2 loop
    dbms_output.put_line(i);
  end loop;
end;
/
```

Output

```SQL
1
2

PL/SQL procedure successfully completed.

SQL>
```

... or defining the datatype with precision (increment is still 1):

Syntax

```SQL
begin
  for i number(5,1) in 1.2 .. 2.2 loop
    dbms_output.put_line(i);
  end loop;
end;
/
```

Output

```SQL
1.2
2.2

PL/SQL procedure successfully completed.

SQL>
```

... or defining the datatype with precision and change the iterator:

Syntax

```SQL
begin
  for i number(5,1) in 1.2 .. 2.2 by 0.2 loop
    dbms_output.put_line(i);
  end loop;
end;
/
```

Output

```SQL
1.2
1.4
1.6
1.8
2
2.2

PL/SQL procedure successfully completed.

SQL>
```

# Tags:

[[NewFeatures]] - [[21c]]