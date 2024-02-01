# 第一个练手题目
NKU计网学院必修课编译原理大作业的makefile

[项目地址](https://github.com/shm0214/2023NKUCS-Compilers-Lab)

## 原始makefile
```
SHELL := /bin/bash
SRC_PATH ?= src
INC_PATH += include
BUILD_PATH ?= build
TEST_PATH ?= test/level1-1
OBJ_PATH ?= $(BUILD_PATH)/obj
BINARY ?= $(BUILD_PATH)/compiler
SYSLIB_PATH ?= sysyruntimelibrary

INC = $(addprefix -I, $(INC_PATH))
SRC = $(shell find $(SRC_PATH)  -name "*.cpp")
CFLAGS = -O2 -g -Wall -Werror $(INC)
FLEX ?= $(SRC_PATH)/lexer.l
LEXER ?= $(addsuffix .cpp, $(basename $(FLEX)))
BISON ?= $(SRC_PATH)/parser.y
PARSER ?= $(addsuffix .cpp, $(basename $(BISON)))
SRC += $(LEXER)
SRC += $(PARSER)
OBJ = $(SRC:$(SRC_PATH)/%.cpp=$(OBJ_PATH)/%.o)
PARSERH ?= $(INC_PATH)/$(addsuffix .h, $(notdir $(basename $(PARSER))))

TESTCASE = $(shell find $(TEST_PATH) -name "*.sy")
TESTCASE_NUM = $(words $(TESTCASE))
LLVM_IR = $(addsuffix _std.ll, $(basename $(TESTCASE)))
GCC_ASM = $(addsuffix _std.s, $(basename $(TESTCASE)))
OUTPUT_LAB3 = $(addsuffix .toks, $(basename $(TESTCASE)))
OUTPUT_LAB4 = $(addsuffix .ast, $(basename $(TESTCASE)))
OUTPUT_LAB5 = $(addsuffix .ll, $(basename $(TESTCASE)))
OUTPUT_LAB6 = $(addsuffix .s, $(basename $(TESTCASE)))
OUTPUT_RES = $(addsuffix .res, $(basename $(TESTCASE)))
OUTPUT_BIN = $(addsuffix .bin, $(basename $(TESTCASE)))
OUTPUT_LOG = $(addsuffix .log, $(basename $(TESTCASE)))

.phony:all app run gdb testlab3 testlab4 testlab5 testlab6 testir test clean clean-all clean-test clean-app llvmir gccasm

all:app

$(LEXER):$(FLEX)
	@flex -o $@ $<

$(PARSER):$(BISON)
	@bison -o $@ $< --warnings=error=all --defines=$(PARSERH)

$(OBJ_PATH)/%.o:$(SRC_PATH)/%.cpp
	@mkdir -p $(OBJ_PATH)
	@g++ $(CFLAGS) -c -o $@ $<

$(BINARY):$(OBJ)
	@g++ -O2 -g -o $@ $^

app:$(LEXER) $(PARSER) $(BINARY)

run:app
	@$(BINARY) -o example.s -S example.sy

gdb:app
	@gdb $(BINARY)

$(OBJ_PATH)/lexer.o:$(SRC_PATH)/lexer.cpp
	@mkdir -p $(OBJ_PATH)
	@g++ $(CFLAGS) -c -o $@ $<

$(TEST_PATH)/%.toks:$(TEST_PATH)/%.sy
	@$(BINARY) $< -o $@ -t

$(TEST_PATH)/%.ast:$(TEST_PATH)/%.sy
	@$(BINARY) $< -o $@ -a

$(TEST_PATH)/%.ll:$(TEST_PATH)/%.sy
	@$(BINARY) $< -o $@ -i

$(TEST_PATH)/%_std.ll:$(TEST_PATH)/%.sy
	@clang -x c $< -S -m32 -emit-llvm -o $@

$(TEST_PATH)/%_std.s:$(TEST_PATH)/%.sy
	@arm-linux-gnueabihf-gcc -x c $< -S -o $@

$(TEST_PATH)/%.s:$(TEST_PATH)/%.sy
	@timeout 5s $(BINARY) $< -o $@ -S 2>$(addsuffix .log, $(basename $@))
	@[ $$? != 0 ] && echo "\033[1;31mCOMPILE FAIL:\033[0m $(notdir $<)" || echo "\033[1;32mCOMPILE SUCCESS:\033[0m $(notdir $<)"

llvmir:$(LLVM_IR)

gccasm:$(GCC_ASM)

testlab3:app $(OUTPUT_LAB3)

testlab4:app $(OUTPUT_LAB4)

testlab5:app $(OUTPUT_LAB5)

testlab6:app $(OUTPUT_LAB6)

.ONESHELL:
testir:app
	@success=0
	@for file in $(sort $(TESTCASE))
	do
		IR=$${file%.*}.ll
		LOG=$${file%.*}.log
		BIN=$${file%.*}.bin
		RES=$${file%.*}.res
		IN=$${file%.*}.in
		OUT=$${file%.*}.out
		FILE=$${file##*/}
		FILE=$${FILE%.*}
		timeout 5s $(BINARY) $${file} -o $${IR} -i 2>$${LOG}
		RETURN_VALUE=$$?
		if [ $$RETURN_VALUE = 124 ]; then
			echo -e "\033[1;31mFAIL:\033[0m $${FILE}\t\033[1;31mCompile Timeout\033[0m"
			continue
		else if [ $$RETURN_VALUE != 0 ]; then
			echo -e "\033[1;31mFAIL:\033[0m $${FILE}\t\033[1;31mCompile Error\033[0m"
			continue
			fi
		fi
		clang -o $${BIN} $${IR} $(SYSLIB_PATH)/sylib.c >>$${LOG} 2>&1
		if [ $$? != 0 ]; then
			echo -e "\033[1;31mFAIL:\033[0m $${FILE}\t\033[1;31mAssemble Error\033[0m"
		else
			if [ -f "$${IN}" ]; then
				timeout 2s $${BIN} <$${IN} >$${RES} 2>>$${LOG}
			else
				timeout 2s $${BIN} >$${RES} 2>>$${LOG}
			fi
			RETURN_VALUE=$$?
			FINAL=`tail -c 1 $${RES}`
			[ $${FINAL} ] && echo -e "\n$${RETURN_VALUE}" >> $${RES} || echo "$${RETURN_VALUE}" >> $${RES}
			if [ "$${RETURN_VALUE}" = "124" ]; then
				echo -e "\033[1;31mFAIL:\033[0m $${FILE}\t\033[1;31mExecute Timeout\033[0m"
			else if [ "$${RETURN_VALUE}" = "127" ]; then
				echo -e "\033[1;31mFAIL:\033[0m $${FILE}\t\033[1;31mExecute Error\033[0m"
				else
					diff -Z $${RES} $${OUT} >/dev/null 2>&1
					if [ $$? != 0 ]; then
						echo -e "\033[1;31mFAIL:\033[0m $${FILE}\t\033[1;31mWrong Answer\033[0m"
					else
						success=$$((success + 1))
						echo -e "\033[1;32mPASS:\033[0m $${FILE}"
					fi
				fi
			fi
		fi
	done
	echo -e "\033[1;33mTotal: $(TESTCASE_NUM)\t\033[1;32mAccept: $${success}\t\033[1;31mFail: $$(($(TESTCASE_NUM) - $${success}))\033[0m"
	[ $(TESTCASE_NUM) = $${success} ] && echo -e "\033[5;32mAll Accepted. Congratulations!\033[0m"
	:

.ONESHELL:
test:app
	@success=0
	@for file in $(sort $(TESTCASE))
	do
		ASM=$${file%.*}.s
		LOG=$${file%.*}.log
		BIN=$${file%.*}.bin
		RES=$${file%.*}.res
		IN=$${file%.*}.in
		OUT=$${file%.*}.out
		FILE=$${file##*/}
		FILE=$${FILE%.*}
		timeout 5s $(BINARY) $${file} -o $${ASM} -S 2>$${LOG}
		RETURN_VALUE=$$?
		if [ $$RETURN_VALUE = 124 ]; then
			echo -e "\033[1;31mFAIL:\033[0m $${FILE}\t\033[1;31mCompile Timeout\033[0m"
			continue
		else if [ $$RETURN_VALUE != 0 ]; then
			echo -e "\033[1;31mFAIL:\033[0m $${FILE}\t\033[1;31mCompile Error\033[0m"
			continue
			fi
		fi
		arm-linux-gnueabihf-gcc -mcpu=cortex-a72 -o $${BIN} $${ASM} $(SYSLIB_PATH)/libsysy.a >>$${LOG} 2>&1
		if [ $$? != 0 ]; then
			echo -e "\033[1;31mFAIL:\033[0m $${FILE}\t\033[1;31mAssemble Error\033[0m"
		else
			if [ -f "$${IN}" ]; then
				timeout 2s qemu-arm -L /usr/arm-linux-gnueabihf $${BIN} <$${IN} >$${RES} 2>>$${LOG}
			else
				timeout 2s qemu-arm -L /usr/arm-linux-gnueabihf $${BIN} >$${RES} 2>>$${LOG}
			fi
			RETURN_VALUE=$$?
			FINAL=`tail -c 1 $${RES}`
			[ $${FINAL} ] && echo -e "\n$${RETURN_VALUE}" >> $${RES} || echo "$${RETURN_VALUE}" >> $${RES}
			if [ "$${RETURN_VALUE}" = "124" ]; then
				echo -e "\033[1;31mFAIL:\033[0m $${FILE}\t\033[1;31mExecute Timeout\033[0m"
			else if [ "$${RETURN_VALUE}" = "127" ]; then
				echo -e "\033[1;31mFAIL:\033[0m $${FILE}\t\033[1;31mExecute Error\033[0m"
				else
					diff -Z $${RES} $${OUT} >/dev/null 2>&1
					if [ $$? != 0 ]; then
						echo -e "\033[1;31mFAIL:\033[0m $${FILE}\t\033[1;31mWrong Answer\033[0m"
					else
						success=$$((success + 1))
						echo -e "\033[1;32mPASS:\033[0m $${FILE}"
					fi
				fi
			fi
		fi
	done
	echo -e "\033[1;33mTotal: $(TESTCASE_NUM)\t\033[1;32mAccept: $${success}\t\033[1;31mFail: $$(($(TESTCASE_NUM) - $${success}))\033[0m"
	[ $(TESTCASE_NUM) = $${success} ] && echo -e "\033[5;32mAll Accepted. Congratulations!\033[0m"
	:

clean-app:
	@rm -rf $(BUILD_PATH) $(PARSER) $(LEXER) $(PARSERH)

clean-test:
	@rm -rf $(OUTPUT_LAB3) $(OUTPUT_LAB4) $(OUTPUT_LAB5) $(OUTPUT_LAB6) $(OUTPUT_LOG) $(OUTPUT_BIN) $(OUTPUT_RES) $(LLVM_IR) $(GCC_ASM) *.toks *.ast *.ll *.s *.out

clean-all:clean-test clean-app

clean:clean-all
```
## 逐步分析
### 定义变量
```
SHELL := /bin/bash
SRC_PATH ?= src
INC_PATH += include
BUILD_PATH ?= build
TEST_PATH ?= test/level1-1
OBJ_PATH ?= $(BUILD_PATH)/obj
BINARY ?= $(BUILD_PATH)/compiler
SYSLIB_PATH ?= sysyruntimelibrary
```
其意义基本如下
- SHELL := /bin/bash：指定Make使用的Shell，而不是默认的/bin/sh。
- SRC_PATH ?= src：源代码目录。
- INC_PATH += include：头文件目录。
- BUILD_PATH ?= build：最后生成的目录。
- TEST_PATH ?= test/level1-1：测试用例目录。
- OBJ_PATH ?= $(BUILD_PATH)/obj：目标文件目录。
- BINARY ?= $(BUILD_PATH)/compiler：输出的二进制可执行文件。
- SYSLIB_PATH ?= sysyruntimelibrary：系统运行库的路径。

```
INC = $(addprefix -I, $(INC_PATH))
SRC = $(shell find $(SRC_PATH)  -name "*.cpp")
CFLAGS = -O2 -g -Wall -Werror $(INC)
FLEX ?= $(SRC_PATH)/lexer.l
LEXER ?= $(addsuffix .cpp, $(basename $(FLEX)))
BISON ?= $(SRC_PATH)/parser.y
PARSER ?= $(addsuffix .cpp, $(basename $(BISON)))
SRC += $(LEXER)
SRC += $(PARSER)
OBJ = $(SRC:$(SRC_PATH)/%.cpp=$(OBJ_PATH)/%.o)
PARSERH ?= $(INC_PATH)/$(addsuffix .h, $(notdir $(basename $(PARSER))))

```
- `INC = $(addprefix -I, $(INC_PATH))`：给头文件加前缀，指定一个被包含makefile的搜索目标。可以使用多个“-I”参数来指定多个目录。
- `SRC = $(shell find $(SRC_PATH) -name "*.cpp")`：在源目录中查找所有源文件。
- `CFLAGS = -O2 -g -Wall -Werror $(INC)`：gcc编译器的标志。
- `FLEX ?= $(SRC_PATH)/lexer.l`：Flex输入文件。
- `LEXER ?= $(addsuffix .cpp, $(basename $(FLEX)))`：生成的词法分析器源文件。
- `BISON ?= $(SRC_PATH)/parser.y`：Bison输入文件。
- `PARSER ?= $(addsuffix .cpp, $(basename $(BISON)))`：生成的语法分析器源文件。
- `SRC += $(LEXER) $(PARSER)`：将词法分析器和语法分析器追加到源文件列表。
- `OBJ = $(SRC:$(SRC_PATH)/%.cpp=$(OBJ_PATH)/%.o)`：目标文件。
- `PARSERH ?= $(INC_PATH)/$(addsuffix .h, $(notdir $(basename $(PARSER))))`:生成的语法分析器头文件
这里最复杂的最后一句是这样的：
先取不是后缀的部分，然后去掉目录部分，再加上.h后缀，最后和目录拼起来。

```
TESTCASE = $(shell find $(TEST_PATH) -name "*.sy")
TESTCASE_NUM = $(words $(TESTCASE))
LLVM_IR = $(addsuffix _std.ll, $(basename $(TESTCASE)))
GCC_ASM = $(addsuffix _std.s, $(basename $(TESTCASE)))
OUTPUT_LAB3 = $(addsuffix .toks, $(basename $(TESTCASE)))
OUTPUT_LAB4 = $(addsuffix .ast, $(basename $(TESTCASE)))
OUTPUT_LAB5 = $(addsuffix .ll, $(basename $(TESTCASE)))
OUTPUT_LAB6 = $(addsuffix .s, $(basename $(TESTCASE)))
OUTPUT_RES = $(addsuffix .res, $(basename $(TESTCASE)))
OUTPUT_BIN = $(addsuffix .bin, $(basename $(TESTCASE)))
OUTPUT_LOG = $(addsuffix .log, $(basename $(TESTCASE)))
```
- `TESTCASE = $(shell find $(TEST_PATH) -name "*.sy")`：查找所有测试用例。
后边是基于测试用例的各种输出文件的变量，有评分的LLVM和GCC还有各个实验自己的。
## 目标
```
.phony:all app run gdb testlab3 testlab4 testlab5 testlab6 testir test clean clean-all clean-test clean-app llvmir gccasm

all:app

$(LEXER):$(FLEX)
	@flex -o $@ $<

$(PARSER):$(BISON)
	@bison -o $@ $< --warnings=error=all --defines=$(PARSERH)

$(OBJ_PATH)/%.o:$(SRC_PATH)/%.cpp
	@mkdir -p $(OBJ_PATH)
	@g++ $(CFLAGS) -c -o $@ $<

$(BINARY):$(OBJ)
	@g++ -O2 -g -o $@ $^

app:$(LEXER) $(PARSER) $(BINARY)

run:app
	@$(BINARY) -o example.s -S example.sy

gdb:app
	@gdb $(BINARY)

$(OBJ_PATH)/lexer.o:$(SRC_PATH)/lexer.cpp
	@mkdir -p $(OBJ_PATH)
	@g++ $(CFLAGS) -c -o $@ $<

$(TEST_PATH)/%.toks:$(TEST_PATH)/%.sy
	@$(BINARY) $< -o $@ -t

$(TEST_PATH)/%.ast:$(TEST_PATH)/%.sy
	@$(BINARY) $< -o $@ -a

$(TEST_PATH)/%.ll:$(TEST_PATH)/%.sy
	@$(BINARY) $< -o $@ -i

$(TEST_PATH)/%_std.ll:$(TEST_PATH)/%.sy
	@clang -x c $< -S -m32 -emit-llvm -o $@

$(TEST_PATH)/%_std.s:$(TEST_PATH)/%.sy
	@arm-linux-gnueabihf-gcc -x c $< -S -o $@

$(TEST_PATH)/%.s:$(TEST_PATH)/%.sy
	@timeout 5s $(BINARY) $< -o $@ -S 2>$(addsuffix .log, $(basename $@))
	@[ $$? != 0 ] && echo "\033[1;31mCOMPILE FAIL:\033[0m $(notdir $<)" || echo "\033[1;32mCOMPILE SUCCESS:\033[0m $(notdir $<)"

llvmir:$(LLVM_IR)

gccasm:$(GCC_ASM)

testlab3:app $(OUTPUT_LAB3)

testlab4:app $(OUTPUT_LAB4)

testlab5:app $(OUTPUT_LAB5)

testlab6:app $(OUTPUT_LAB6)
```

- `.phony: all app run gdb testlab3 testlab4 testlab5 testlab6 testir test clean clean-all clean-test clean-app llvmir gccasm`：伪目标，这样所有的命令都要指定才会允许。
- `all: app`：默认目标是构建app。
- `app: $(LEXER) $(PARSER) $(BINARY)`：构建编译器应用程序。
- `run: app`：在样例输入（example.sy）上运行编译器。
- `gdb: app`：在编译的二进制文件上运行GDB。

这里的自动化变量很重要，`$<`是依赖的挨个的值，而`$@`是目标的挨个的值，举例
```
$(TEST_PATH)/%_std.s:$(TEST_PATH)/%.sy
	@arm-linux-gnueabihf-gcc -x c $< -S -o $@
```
这里就是把所有在测试目录底下每一个叫.sy的文件视为C语言(-x c)，然后编译成汇编(-S)，输出在这个目录底下同一个名字但是叫_std.s的汇编文件(-o)
## 函数
```
.ONESHELL:
testir:app
	@success=0
	@for file in $(sort $(TESTCASE))
	do
		IR=$${file%.*}.ll
		LOG=$${file%.*}.log
		BIN=$${file%.*}.bin
		RES=$${file%.*}.res
		IN=$${file%.*}.in
		OUT=$${file%.*}.out
		FILE=$${file##*/}
		FILE=$${FILE%.*}
		timeout 5s $(BINARY) $${file} -o $${IR} -i 2>$${LOG}
		RETURN_VALUE=$$?
		if [ $$RETURN_VALUE = 124 ]; then
			echo -e "\033[1;31mFAIL:\033[0m $${FILE}\t\033[1;31mCompile Timeout\033[0m"
			continue
		else if [ $$RETURN_VALUE != 0 ]; then
			echo -e "\033[1;31mFAIL:\033[0m $${FILE}\t\033[1;31mCompile Error\033[0m"
			continue
			fi
		fi
		clang -o $${BIN} $${IR} $(SYSLIB_PATH)/sylib.c >>$${LOG} 2>&1
		if [ $$? != 0 ]; then
			echo -e "\033[1;31mFAIL:\033[0m $${FILE}\t\033[1;31mAssemble Error\033[0m"
		else
			if [ -f "$${IN}" ]; then
				timeout 2s $${BIN} <$${IN} >$${RES} 2>>$${LOG}
			else
				timeout 2s $${BIN} >$${RES} 2>>$${LOG}
			fi
			RETURN_VALUE=$$?
			FINAL=`tail -c 1 $${RES}`
			[ $${FINAL} ] && echo -e "\n$${RETURN_VALUE}" >> $${RES} || echo "$${RETURN_VALUE}" >> $${RES}
			if [ "$${RETURN_VALUE}" = "124" ]; then
				echo -e "\033[1;31mFAIL:\033[0m $${FILE}\t\033[1;31mExecute Timeout\033[0m"
			else if [ "$${RETURN_VALUE}" = "127" ]; then
				echo -e "\033[1;31mFAIL:\033[0m $${FILE}\t\033[1;31mExecute Error\033[0m"
				else
					diff -Z $${RES} $${OUT} >/dev/null 2>&1
					if [ $$? != 0 ]; then
						echo -e "\033[1;31mFAIL:\033[0m $${FILE}\t\033[1;31mWrong Answer\033[0m"
					else
						success=$$((success + 1))
						echo -e "\033[1;32mPASS:\033[0m $${FILE}"
					fi
				fi
			fi
		fi
	done
	echo -e "\033[1;33mTotal: $(TESTCASE_NUM)\t\033[1;32mAccept: $${success}\t\033[1;31mFail: $$(($(TESTCASE_NUM) - $${success}))\033[0m"
	[ $(TESTCASE_NUM) = $${success} ] && echo -e "\033[5;32mAll Accepted. Congratulations!\033[0m"
	:
```
>借助一下chatgpt
1. **`.ONESHELL:`：**
   - 这是一个Makefile特殊的标记，它告诉Make工具在一个Shell中执行整个规则的命令。这样可以确保整个规则中的命令共享相同的Shell环境。

2. **`testir:app`：**
   - 这定义了一个名为 `testir` 的目标，它依赖于 `app`。也就是说，在执行 `testir` 之前，确保 `app` 已经构建。

3. **命令部分：**
   - 代码段以 `@success=0` 开始，初始化一个计数器 `success` 为 0。

   - 通过循环遍历所有测试用例文件 `$(TESTCASE)`：

      - `IR=$${file%.*}.ll`：定义 `IR` 变量，表示将测试用例文件的扩展名 `.sy` 替换为 `.ll`。

      - `LOG=$${file%.*}.log`：定义 `LOG` 变量，表示日志文件。

      - `BIN=$${file%.*}.bin`：定义 `BIN` 变量，表示生成的二进制文件。

      - `RES=$${file%.*}.res`：定义 `RES` 变量，表示执行结果输出文件。

      - `IN=$${file%.*}.in`：定义 `IN` 变量，表示输入文件。

      - `OUT=$${file%.*}.out`：定义 `OUT` 变量，表示期望输出文件。

      - `FILE=$${file##*/}`：提取测试用例文件的纯文件名，去掉路径部分。

      - `FILE=$${FILE%.*}`：去掉文件名的扩展名，得到测试用例的基本名。

      - `timeout 5s $(BINARY) $${file} -o $${IR} -i 2>$${LOG}`：运行编译器，生成 LLVM IR。`-o $${IR}` 指定输出文件，`-i` 表示生成 LLVM IR。

      - `RETURN_VALUE=$$?`：获取上一条命令的返回值。

      - 条件判断，根据返回值的不同情况输出相应的信息，例如编译超时、编译错误等。

      - `clang -o $${BIN} $${IR} $(SYSLIB_PATH)/sylib.c >>$${LOG} 2>&1`：使用 `clang` 将 LLVM IR 编译成二进制文件。

      - 进行测试执行，检查执行结果，并根据情况输出相应的信息。

      - 最后统计成功和失败的测试数目。

4. **`echo -e "\033[1;33mTotal: $(TESTCASE_NUM)\t\033[1;32mAccept: $${success}\t\033[1;31mFail: $$(($(TESTCASE_NUM) - $${success}))\033[0m"`：**
   - 输出测试总数、通过数、失败数的统计信息。

5. **`[ $(TESTCASE_NUM) = $${success} ] && echo -e "\033[5;32mAll Accepted. Congratulations!\033[0m"`：**
   - 如果所有测试都通过，则输出祝贺消息。

6. **`:`：**
   - 这是一个单独的冒号，表示一个空操作。在这里，它用于确保目标 `testir` 总是被认为是 "up to date"，即不需要重新构建。

## 清除
```

clean-app:
	@rm -rf $(BUILD_PATH) $(PARSER) $(LEXER) $(PARSERH)

clean-test:
	@rm -rf $(OUTPUT_LAB3) $(OUTPUT_LAB4) $(OUTPUT_LAB5) $(OUTPUT_LAB6) $(OUTPUT_LOG) $(OUTPUT_BIN) $(OUTPUT_RES) $(LLVM_IR) $(GCC_ASM) *.toks *.ast *.ll *.s *.out

clean-all:clean-test clean-app

clean:clean-all
```
清理对应的目录，注意clean也依赖了多个伪目标