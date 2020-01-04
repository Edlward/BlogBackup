---
title: Nand Flash编程
date: 2019-12-27 20:39:46
img: http://q3996b08i.bkt.clouddn.com/embedded-study/img_cirtuit.jpg
top: false
toc: true
summary: S3C2440 Nand Flash编程
categories: S3C2440
tags: 
	- Nand
	- ARM9
---

## 1.读芯片ID

### 1.1 读芯片ID时序

![](http://q3996b08i.bkt.clouddn.com/embedded-study/nand_timing.png)

 简化为4个步骤：
 - 1.使能片选
 - 2.写命令0x90
 - 3.写地址0x00
 - 4.读ID信息
 ```c
/* 识别NAND FLASH */
void scan_nand_flash(void)
{
	int i;
	//保存读取ID信息的数组
	unsigned char id_info[5] = {0};

	nand_enable_cs();//使能CS
	nand_write_cmd(0x90);
	nand_write_addr(0x00);

	for(i = 0;i < 5;i++){
		id_info[i] = nand_read_data();
	}
	nand_disable_cs();//禁止CS
    
	printf("Maker  Code: 0x%x\r\n",id_info[0]);
	printf("Device Code: 0x%x\r\n",id_info[1]);
	printf("3rd   cycle: 0x%x\n\r",id_info[2]);
	printf("Page   size: %d KB\n\r",1  << (id_info[3] & 0x03));//页大小与id_info[3]最低2位有关
	printf("Block  size: %d KB\n\r",64 << ((id_info[3] >> 4) & 0x03));//块大小与id_info[3]第4、5位有关

	printf("5th   cycle: 0x%x\n\r",id_info[4]);
```
### 1.2 由ID数据获得芯片参数
![](http://q3996b08i.bkt.clouddn.com/embedded-study/nand_scan_test.png)


![](http://q3996b08i.bkt.clouddn.com/embedded-study/nand_chip_params.png)

![](http://q3996b08i.bkt.clouddn.com/embedded-study/nand_scan_test2.png)

- ID信息的第4字节为`0x95`
- 页大小与id_info[3]最低2位有关，可得**页大小**为：`2KB`
- 块大小与id_info[3]第4、5位有关，可得**块大小**为：`128KB`


> 注意：如果此时烧写到Nand Flash，并从Nand Flash启动程序是不会成功的，因为这个bin文件大小已经超过了4K，且现在还没有实现nand flash的读函数。

## 2.读数据
> 目标：实现从NAND FLASH中启动，重定位所有数据至SDRAM，并实现读取芯片ID数据

### 2.1 NAND内部结构分析
![](http://q3996b08i.bkt.clouddn.com/embedded-study/nand_struct.png)

- OOB：out of bank（在bank之外）
由上图可得：
- 1Page = 2KB + 64B
- 1Block = 64 * Pages = 128KB + 4KB
- 1Device = 2048 * Blocks = 256MB + 8MB


![](http://q3996b08i.bkt.clouddn.com/embedded-study/nand_data_store.png)

- OOB区的作用：因为nand的缺点是会发生`“位反转”`，为了解决这个问题，nand中的OOB区，**用于校验数据区的数据是否发生错误**，当有错误时，可以恢复数据。（其本身不存储数据）
- 因为OOB中并不存放数据，只是用于校验数据区是否发生错误，因此当CPU读取Nand Flash**第2048**个数据，该数据为 `Page1中的第0个byte`


### 2.2 地址序列与时序

![](http://q3996b08i.bkt.clouddn.com/embedded-study/nand_addr.png)
- 由地址序列可以看出：发出地址信号共需5个周期，前2个周期发出列地址（Column Address），后3个周期发出行地址（Row Address）
- 地址线序列有一些位是没有用到的，其目的也是以后兼容更大芯片的nand falsh
![](http://q3996b08i.bkt.clouddn.com/embedded-study/nand_read_timing.png)


> Nand Flash内部结构展开大致如下：
![](http://q3996b08i.bkt.clouddn.com/embedded-study/nand_struct2.png)

### 2.3 读数据流程
- 1.发出片选信号
- 2.发出0x00命令
- 3.**发送5个周期的地址（两个列地址，三个行地址（page））**
- 4.再发送0x30命令
- 5.等待就绪
- 6.读数据
- 7.禁止片选

### 2.4 转换所读地址的列与页
将输入的地址addr转换为：
- 列地址（Col Address）
    - page是定位到哪一个页，col变量定位的就是在这个页的偏移量（在这个页上的第几列0~2047）
- 行地址（页）
    - 因为读取数据的时候是一次性读出一页，因此当给出地址addr之后，每一页的数据大小是2K，因此我们可以根据地址知道我们读取的数据是哪一个页

```c
	int col  = addr % 2048;//列地址  addr &(2048-1); 

	int page = addr / 2048;//行地址（相当于页地址）
```


### 2.5 NAND等待就绪
![](http://q3996b08i.bkt.clouddn.com/embedded-study/nand_nfcon.png)

- 当NFSATA[0] = 0时，表示正忙
- 当NFSATA[0] = 1时，表示就绪
```c
/* 等待NAND就绪 */
void nand_wait_ready(void)
{
	while(!(NFSTAT & 0x01));//当NFSATA[0] = 1时，表示就绪
}
```

### 2.6 读取数据函数
```c
/* NAND FLASH读取数据
 * param:读取的地址、存放的地址、读取的长度
 */
void read_nand_data(unsigned int addr,unsigned char *buf,unsigned int len)
{
	int i = 0;

	/*page是定位到哪一个页，col变量定位的就是在这个
	 *页的偏移量（在这个页上的第几列0~2047）
	 */
	int col  = addr % 2048;//列地址  addr &(2048-1); 

    /* 因为读取数据的时候是一次性读出一页，因此当给出
	 * 地址addr之后，每一页的数据大小是2K，因此我们可以
	 * 根据地址知道我们读取的数据是哪一个页
	 */
	int page = addr / 2048;//行地址（相当于页地址）

	nand_enable_cs();//1.使能CS

	while(i < len){
		nand_write_cmd(0x00);       //2.发出0x00命令

		/* col addr */
		nand_write_addr(col & 0xFF);//3.发出地址
		nand_write_addr((col >> 8) & 0xFF);  

 		/* row/page addr */
	    nand_write_addr(page & 0xFF);
	    nand_write_addr((page >>  8) & 0xFF);
		nand_write_addr((page >> 16) & 0xFF);	

		nand_write_cmd(0x30);//4.发出0x30命令

        nand_wait_ready(); //5.等待就绪
        /* for循环中有2个条件
         * 1.当读到页尾，但还是没有读完，说明需要读取下一页
         * 2.当已经读取指定字节数，则不再读取
         */     
        for(; (col < 2048) && (i < len); col++){//6.读数据
        	buf[i++] = nand_read_data_byte();
        }
        if(i == len){
        	break;
        }
        col = 0;
        page++;//指向下一页
	}
	nand_disable_cs(); //7.禁止CS
}
```

### 2.7 NAND重定位
> 从Nand Flash启动，此时片内SRAM的地址对应的就是CPU的0地址，如果从Nand Flash启动，2440硬件会把nand Flash前4K的数据复制到片内SRAM,如果Nand Flash上的程序大于4K,那后续数据就会丢失，相当于只重定位了前4K的代码。

如何解决上述问题：
- 1.前提：实现了NAND FLASH读取数据函数
- 2.代码烧写到NAND FLASH,并从NAND中启动
- 3.程序运行到重定位代码的位置判断一下，是从Nand Flash启动还是NOR  Flash启动（通过往0地址写数据，因为Nand是支持读写的，所以读出的结果和写的结果一样，而NOR Flash不能像内存一样读写，因此读写的内容是不一致的）
- 4.如果从NOR Flash启动，直接使用简单的重定位代码就行，如果是Nand Flash启动，那就是用Nnad Flash的读函数进行代码的重定位。

---

> 首先判断从NorFlash or NandFlash中启动
```c
/* 检查是否从NorFlash中启动
 * 方法：写0x12345678到0地址，在读取出来，如果得到0x12345678，表示0地址上的内容被修改，即为片内RAM，则为nand启动
 * 原因：原因：nor不能直接写入，写入需要发出一定格式的数据，才能写入
 * 返回0为nand启动，返回1为nor启动
*/
int isBootFromNorFlash(void)
{
	volatile unsigned int *p = (volatile unsigned int *)0;
	unsigned int val = *p;//暂存[0]上的数据
	*p = 0xdeadc0de;//dead code任意值
	if(0xdeadc0de == *p){
		/* 写成功，对应nand启动 */
		*p = val;//恢复原来的值
		return 0;
	}
	else{
		return 1;
	}
}
```

> 重定位代码
```c
/*
 * 将除bss段的全部数据拷贝到sdram中
 * 传递形参，原地址src:_start  目标地址dest:__bss_start  长度len:__bss_start-_star
 */
void copy_to_sdram(void)
{
	/* 要从lds文件中获取__code_start、__bss_start
	 * 然后从0地址把数据复制到__code_start
	 */
	extern int __code_start,__bss_start; //声明外部变量

	volatile unsigned int *src  = (volatile unsigned int *)0; //flash中0地址 
	volatile unsigned int *dest = (volatile unsigned int *)&__code_start; //目标地址：sdram中的0x30000000地址
	volatile unsigned int *end  = (volatile unsigned int *)&__bss_start;  //结束地址：bss的起始地址

	int len = (int)&__bss_start - (int)&__code_start;//获取数据总长度

	if(isBootFromNorFlash()){//如果从Nor中启动
		while(dest < end){
			*dest++ = *src++; //拷贝
		}
	}
	else{//从Nand中启动,需要先初始化nand，然后重定位代码
		nand_init();
        //从 src 复制到 des ,总共复制len字节，也就是重定位的代码
		read_nand_data((unsigned int)src,(unsigned char *)dest,len);
	}
}
```
### 2.7 读数据测试
> 读取0地址后160bytes的数据，如果跟.bin文件前160字节数据相同，则读取成功，否则读取失败
```c
/* 测试函数：读数nand上160bytes数据 
 */
void read_nand_flash(void)
{
	int i,j;
	unsigned int addr,hex_addr;
	unsigned char c,str[16],data[160];
	volatile unsigned char *p;

	/* 获得地址 */
	printf("*****Enter the address to read:");
	addr = get_uint();

	read_nand_data(addr,data,160);//获取到地址上的数据
	p = (volatile unsigned char *)data;//p指向data,用于打印data数据

	hex_addr = addr;//起始地址
	printf("Read Data:\r\n");
	printf("            00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f\n\r");
 	/* 长度固定为160 bytes */	
	for(i = 0; i < 10; i++){
		printf("0x%08x  ",hex_addr);
		//每行打印16个16进制数
		for(j = 0; j < 16; j++){
			c = *p++;//读取16个
			str[j] = c;//保存字符
			//先打印数值
			printf("%02x ",c);
		}
		printf(" | ");
		//后打印字符
		for(j = 0; j < 16; j++){
			if(str[j] < 0x20 || str[j] > 0x7e){//不可视字符,打印‘.’
				printf(".");
			}
			else{
				printf("%c",str[j]);
			}
		}
		hex_addr+=16;//换行+16
		printf("\n\r");
	}

}
```
![](http://q3996b08i.bkt.clouddn.com/embedded-study/nand_bin_data.png)

![](http://q3996b08i.bkt.clouddn.com/embedded-study/nand_read_test2.png)
由上测试可知读取成功


## 3.擦除

### 3.1 擦除时序
- 1.发出片选信号
- 2.发出0x60命令
- 3.**发送3个行地址（page）**
- 4.再发送0xD0命令
- 5.等待就绪（等待擦除完成）
- 6.禁止片选

![](http://q3996b08i.bkt.clouddn.com/embedded-study/nand_erase_timing.png)


### 3.2 地址和参数合法性
- 由于按块来擦除（128KB），因此地址和参数都必须是128K的倍数
- 
```c
	/* 检查参数合法性 */
	if(addr & (0x1FFFF)){ //128K
		printf("Err addr! Please enter an integral multiple of 128K\r\n");
		return -1;
	}
	if(len & (0x1FFFF)){ //128K
		printf("Err len! Please enter an integral multiple of 128K\r\n");
		return -1;
	}
```

### 3.3 擦除函数
```c
/* NAND FLASH擦除数据
 * param:擦除的起始地址、擦除的长度(byte)
 * ret:失败-1，成功0
*/
int erase_nand_data(unsigned int addr,unsigned int len)
{
	int page = addr / 2048;//行地址

	/* 检查参数合法性 */
	if(addr & (0x1FFFF)){ //128K
		printf("Err addr! Please enter an integral multiple of 128K\r\n");
		return -1;
	}
	if(len & (0x1FFFF)){ //128K
		printf("Err len! Please enter an integral multiple of 128K\r\n");
		return -1;
	}
	/* 1.使能CS */
	nand_enable_cs();
	while(1)
	{
		page = addr / 2048;//行地址

		/* 2.发出0x60命令 */
		nand_write_cmd(0x60); 
		/* 3.发出地址row/page addr */
		nand_page(page);      
		/* 4.发出0xD0命令 */
		nand_write_cmd(0xD0); 
		/* 5.等待就绪(等待擦除完成) */
		nand_wait_ready();    

		len -= (128*1024);//长度减去一个block
		if(0 == len){
			break;
		}
		addr += (128*1024);//指向下一块
	} 
	/* 6.禁止CS */
	nand_disable_cs();
	return 0;
}
```

### 3.4 擦除数据测试
```c
/* 擦除测试函数：固定擦除一个1block = 128K
*/
void erase_nand_flash(void)
{
	int addr;
	unsigned int whichblock;
	/* 获得第几个Block */
	printf("Enter the address of sector to erase: ");
	addr = get_uint();

	whichblock = addr / (128*1024);

	printf("***** block number : [ %d ]\r\n",whichblock);

	/* 提示擦除数据的范围 */
	printf("***** Erase range : 0x%08x - 0x%08x\n\r",addr,(addr+(128*1024)));
 	printf("***** erase ...\r\n");

 	if(erase_nand_data(addr,128*1024) == 0){//如果擦除成功
 		printf("***** Erase finished!\r\n");
 	}
 	else{
 		printf("***** Erase fail!\r\n");
 	}
}
```
![](http://q3996b08i.bkt.clouddn.com/embedded-study/nand_erase_test.png)

## 4.写数据 

### 4.1 写数据时序
- 1.发出片选信号
- 2.发出0x80命令
- 3.**发送5个周期的地址（两个列地址，三个行地址（page））**
- 4.写入数据
- 5.再发送0x10命令
- 6.等待就绪（等待擦除完成）
- 7.禁止片选
![](http://q3996b08i.bkt.clouddn.com/embedded-study/nand_wriet_timing.png)


### 4.2 写数据函数
```c
/* NAND FLASH写入数据
 * param:写入的地址、数据指针、写入的长度
 */
void write_nand_data(unsigned int addr,unsigned char *buf,unsigned int len)
{
	int i = 0;
	int page = addr / 2048;
	int col = addr & (2048 - 1);
	/* 1.使能CS */
	nand_enable_cs();
	while(1){
		/* 2.发出0x80命令 */
		nand_write_cmd(0x80);
		/* 3.发出地址 */
		nand_col(col);
		nand_page(page);
		/* 4.写入数据*/
		for(; (col < 2048) && (i < len); ){
			nand_write_data_byte(buf[i++]);			
		}
		/* 5.发出0x10命令 */
		nand_write_cmd(0x10);
		/* 6.等待就绪(等待写入完成) */
		nand_wait_ready();
		if(i == len){
			break;
		}
		else{
			col = 0;
			page++;
		}

	}
	/* 7.禁止CS */
	nand_disable_cs();
}
```


### 4.3 写数据测试
> 此处注意：一般在烧写数据之前需要对数据进行擦除操作，除非原本的数据全f，否则都需要进行擦除，不然写入的数据会有问题。
```c
void write_nand_flash(void)
{
	unsigned int addr;
	unsigned char str[50];
	unsigned int len;

	/* 获得第几个Block */
	printf("***** Enter addr to write: ");
	addr = get_uint();

	printf("***** Enter the string to write: ");
	gets(str);
	len = strlen(str) + 1;

	printf("***** write range : 0x%08x - 0x%08x\r\n",addr,(addr + len));
	printf("***** writing ...\r\n");

	write_nand_data(addr,str,strlen(str)+1);//strlen不包括结束符'\0'，因此需要+1
	printf("***** writing finished\r\n");
}
```
- 在`0x100000`地址写入“hello,world!”
![](http://q3996b08i.bkt.clouddn.com/embedded-study/nand_write_test.png)
- 读取数据
![](http://q3996b08i.bkt.clouddn.com/embedded-study/nand_read_test3.png)


## 5.判断是否为坏块
![](http://q3996b08i.bkt.clouddn.com/embedded-study/nand_data_store.png)

> 通过读取OOB区的第0个字节（即第`2048`个字节）来判断，如果不是0xFF，为坏块，否则不是

```c
/* 判断是否为坏块
 * ret:返回1为坏块，返回0不是
 */
int isNandBadBlock(unsigned int addr)
{
	unsigned int col = 2048;//读取OOB区第0个字节
	unsigned int page = addr / 2048;
	unsigned char val = 0;
	/* 1. 选中 */
	nand_enable_cs();
	/* 2. 发出读命令00h */
	nand_write_cmd(0x00);
	/* 3. 发出地址(分5步发出) */
	nand_col(col);
	nand_page(page);
	/* 4. 发出读命令30h */
	nand_write_cmd(0x30);
	/* 5. 判断状态,等待就绪 */
	nand_wait_ready();
	/* 6. 读数据 */
	val = nand_read_data_byte();
	/* 7. 取消选中 */	
	nand_disable_cs();

	if(val != 0xFF){
		return 1; /* 坏块 */
	}
	else{
		return 0;
	}
}
```
## 6.ECC
- ECC（Error Checking and Correction），是一种用于Nand的差错检测和修正算法。