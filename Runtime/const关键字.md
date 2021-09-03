1、声明常变量，使得指定的变量不能被修改。

```c
const int a = 5; /* a的值一直为5，不能被改变 */ 

const int b;  
b = 10; 

/* b的值被赋值为10后，不能被改变 */
	
const int *ptr; /* ptr为指向整型常量的指针，ptr的值可以修改，但不能修改其所指向的值 */ 

int *const ptr; /*ptr为指向整型的常量指针，ptr的值不能修改，但可以修改其所指向的值 */ 

const int *const ptr; /*ptr为指向整型常量的常量指针，ptr及其指向的值都不能修改 */

```

2、修饰函数形参，使得形参在函数内不能被修改，表示输入参数。如下:

```c
int fun(const int a);
//或
int fun(const char *str);
```

3、修饰函数返回值，使得函数的返回值不能被修改。

```c

const char *getstr(void);
//使用：
const *str= getstr(); 

const int getint(void); 
//使用：
const int a = getint();

```