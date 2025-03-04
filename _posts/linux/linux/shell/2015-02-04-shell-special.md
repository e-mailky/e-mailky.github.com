---
layout: post
title:  "Linux下高效编写Shell——Shell特殊字符汇总"
date:   2015-2-4 10:09:35
categories: [Linux]
tags: [Linux, Shell, ]
description: ""
---
Linux下无论如何都是要用到shell命令的，在Shell的实际使用中，有编程经验的很容易上手，但稍微有难度的是shell里面的那些个符号，各种特殊的符号在我们编写Shell脚本的时候如果能够用的好，往往能给我们起到事半功倍的效果，为此，特地将Shell里面的一些符号说明罗列成对照表的形式，以便快速的查找。看看你知道下表中哦你的哪些Shell符号呢？

Shell符号及各种解释对照表：
<table class="table table-bordered" style="border-collapse:separate;max-width:100%;border-spacing:0px;width:729px;margin-bottom:20px;border-width:1px 1px 1px 0px;border-top-style:solid;border-right-style:solid;border-bottom-style:solid;border-top-color:#DDDDDD;border-right-color:#DDDDDD;border-bottom-color:#DDDDDD;border-top-left-radius:4px;border-top-right-radius:4px;border-bottom-right-radius:4px;border-bottom-left-radius:4px;">
		
			<tr>
				<th style="padding:8px;line-height:20px;vertical-align:bottom;border-top-width:0px;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;border-top-left-radius:4px;">
					<a rel="tag" href="http://blog.useasp.net/tags/shell%E7%AC%A6%E5%8F%B7" style="color:#0088CC;text-decoration:none;">Shell符号</a>
				</th>
				<th style="padding:8px;line-height:20px;vertical-align:bottom;border-top-width:0px;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;border-top-right-radius:4px;">
					使用方法及说明
				</th>
			</tr>
		
		<tbody>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					#
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					<p style="margin-top:0px;margin-bottom:10px;">
						注释符号(Hashmark[Comments])
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						1.在shell文件的行首，作为include标记，#!/bin/bash;
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						2. 其他地方作为注释使用，在一行中，#后面的内容并不会被执行，除非；
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						3. 但是用单/双引号包围时，#作为#号字符本身，不具有注释作用。
					</p>
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					;
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					作为多语句的分隔符(Command separator [semicolon])。多个语句要放在同一行的时候，可以使用分号分隔。注意，有时候分号需要转义。
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					;;
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					连续分号(Terminator [double semicolon])。在使用case选项的时候，作为每个选项的终结符。在Bash version 4+ 的时候，还可以使用[;;&amp;], [;&amp;]
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					.
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					<p style="margin-top:0px;margin-bottom:10px;">
						点号(dot command [period])。
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						1. 相当于bash内建命令source，如：
					</p>
                   <p style="margin-top:0px;margin-bottom:10px;">
						2. 作为文件名的一部分，在文件名的开头，表示该文件为隐藏文件，ls一般不显示出来（ls -a 可以显示）；
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						3. 作为目录名，一个点代表当前目录，两个点号代表上层目录（当前目录的父目录）。注意，两个以上的点不出现，除非你用引号（单/双）包围作为点号字符本身；
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						4. 正则表达式中，点号表示任意一个字符。
					</p>
				</td>
			</tr>
            <tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					"
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					双引号（partial quoting [double quote]）。部分引用。双引号包围的内容可以允许变量扩展，也允许转义字符的存在。如果字符串内出现双引号本身，需要转义，因此不一定双引号是成对的。
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					'
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					单引号(full quoting [single quote])。单引号括住的内容，被视为单一字符串，引号内的禁止变量扩展，所有字符均作为字符本身处理（除单引号本身之外），单引号必须成对出现。
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					,
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					<p style="margin-top:0px;margin-bottom:10px;">
						逗号(comma operator [comma])。
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						1. 用在连接一连串的数学表达式中，这串数学表达式均被求值，但只有最后一个求值结果被返回。如：
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
					    2. 用于参数替代中，表示首字母小写，如果是两个逗号，则表示全部小写，注意，这个特性在bash version 4的时候被添加的。例子：
					</p>
            </td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					\
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					<p style="margin-top:0px;margin-bottom:10px;">
						反斜线，反斜杆(escape [backslash])。
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						1. 放在特殊符号之前，转义特殊符号的作用，仅表示特殊符号本身，这在字符串中常用；
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						2. 放在一行指令的最末端，表示紧接着的回车无效（其实也就是转义了Enter），后继新行的输入仍然作为当前指令的一部分。
					</p>
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					/
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					<p style="margin-top:0px;margin-bottom:10px;">
						斜线，斜杆（Filename path separator [forward slash]）。
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						1.作为路径的分隔符，路径中仅有一个斜杆表示根目录，以斜杆开头的路径表示从根目录开始的路径；
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						2.在作为运算符的时候，表示除法符号。如：
					</p>
                </td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					`
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					反引号，后引号（Command substitution[backquotes])。命令替换。这个引号包围的为命令，可以执行包围的命令，并将执行的结果赋值给变量。如：
                </td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					:
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					<p style="margin-top:0px;margin-bottom:10px;">
						冒号(null command [colon])。空命令，这个命令什么都不做，但是有返回值，返回值为0（即：true）。这个命令的作用非常奇妙。
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						1. 可做while死循环的条件；
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						2. 在if分支中作为占位符（即某一分支什么都不做的时候）；
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						3. 放在必须要有两元操作的地方作为分隔符，如：
					</p>
                    <p style="margin-top:0px;margin-bottom:10px;">
						4. 在参数替换中为字符串变量赋值，在重定向操作(&gt;)中，把一个文件长度截断为0（:&gt;&gt;这样用的时候，目标存在则什么都不做），这个只能在普通文件中使用，不能在管道，符号链接和其他特殊文件中使用；
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						5. 甚至你可以用来注释（#后的内容不会被检查，但:后的内容会被检查，如果有语句如果出现语法错误，则会报错）；
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						6. 你也可以作为域分隔符，比如环境变量$PATH中，或者passwd中，都有冒号的作为域分隔符的存在；
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						7. 你也可以将冒号作为函数名，不过这个会将冒号的本来意义转变（如果你不小心作为函数名，你可以使用unset -f :&nbsp;来取消function的定义）。
					</p>
				</td>
			</tr>
<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					!
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					<p style="margin-top:0px;margin-bottom:10px;">
						感叹号（reverse (or negate) [bang],[exclamation mark])。取反一个测试结果或退出状态。
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						1. 表示反逻辑，比如后面的!=,这个是表示不等于；
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						2. 表示取反，如：ls a[!0-9] #表示a后面不是紧接一个数字的文件；
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						3. 在不同的环境里面，感叹号也可以出现在间接变量引用里面；
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						4. 在命令行中，可以用于历史命令机制的调用，你可以试试!$,!#，或者!-3看看，不过要注意，这点特性不能在脚本文件里面使用（被禁用）。
					</p>
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					*
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					<p style="margin-top:0px;margin-bottom:10px;">
						星号（wildcard/arithmetic operator[asterisk])。
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						1. 作为匹配文件名扩展的一个通配符，能自动匹配给定目录下的每一个文件；
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						2. 正则表达式中可以作为字符限定符，表示其前面的匹配规则匹配任意次；
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						3. 算术运算中表示乘法。
					</p>
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					**
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					双星号(double asterisk)。算术运算中表示求幂运算。
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					?
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					<p style="margin-top:0px;margin-bottom:10px;">
						问号（test operator/wildcard[Question mark])。
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						1. 表示条件测试；
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						2. 在双括号内表示C风格的三元操作符((condition?true-result:false-result))；
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						3. 参数替换表达式中用来测试一个变量是否设置了值；
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						4. 作为通配符，用于匹配文件名扩展特性中，用于匹配单个字符；
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						5. 正则表达式中，表示匹配其前面规则0次或者1次。
					</p>
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					$
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					<p style="margin-top:0px;margin-bottom:10px;">
						美元符号(Variable substitution[Dollar sign])。前面已经表示过一种意思。
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						1. 作为变量的前导符，用作变量替换，即引用一个变量的内容，比如：echo $PATH；
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						2.在正则表达式中被定义为行末（End of line）。
					</p>
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					${}
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					参数替换(Variable substitution)。请参考具体内容以获得的详细信息。
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					$‘...’
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					引用内容展开，执行单引号内的转义内容（单引号原本是原样引用的），这种方式会将引号内的一个或者多个[\]转义后的八进制，十六进制值展开到ASCII或Unicode字符。
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					$*, $@
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					位置参数，这个在使用脚本文件的时候，在传递参数的时候会用到。两者都能返回调用脚本文件的所有参数，但$*是将所有参数作为一个整体返回（字符串），而$@是将每个参数作为单元返回一个参数列表。注意，在使用的时候需要用双引号将$*,$@括住。这两个变量收$IFS的影响，如果在实作，要考虑其中的一些细节，更多请自行查阅参考位置参数(Positional Parameters)的相关信息。
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					$?
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					此变量值在使用的时候，返回的是最后一个命令、函数、或脚本的退出状态码值，如果没有错误则是0，如果为非0，则表示在此之前的最后一次执行有错误。
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					$$
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					进程ID变量，这个变量保存了运行当前脚本的进程ID值。
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					()
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					<p style="margin-top:0px;margin-bottom:10px;">
						圆括号(parentheses)。
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						1， 命令组（Command group）。由一组圆括号括起来的命令是命令组，命令组中的命令实在子shell（subshell）中执行。因为是在子shell内运行，因此在括号外面是没有办法获取括号内变量的值，但反过来，命令组内是可以获取到外面的值，这点有点像局部变量和全局变量的关系，在实作中，如果碰到要cd到子目录操作，并在操作完成后要返回到当前目录的时候，可以考虑使用subshell来处理；
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						2. 用于数组的初始化。
					</p>
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					{xx,yy,zz,...}
				</td>
             <td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					花括号扩展(Brace Expansion)。 在命令中可以用这种扩展来扩展参数列表，命令将会依照列表中的括号分隔开的模式进行匹配扩展。注意的一点是，这花括号扩展中不能有空格存在，如果确实有必要空格，则必须被转义或者使用引号来引用。例子:
</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					{a..z}
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					在Bash version 3时添加了这种花括号扩展的扩展，可以使用{A..Z}表示A-Z的所有字符列表，这种方式的扩展Mitchell测试了一下，好像仅适用于A-Z，a-z，还有数字{minNo..maxNo}的这种方式扩展。
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					{}
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					代码块(curly brackets)。这个是匿名函数，但是又与函数不同，在代码块里面的变量在代码块后面仍能访问。注意：花括号内侧需要有空格与语句分隔。另外，在xargs -i中的话，还可以作为文本的占位符，用以标记输出文本的位置。
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					{} \;
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					这个{}是表示路径名，这个并不是shell内建的，现在接触到的情况看，好像只用在find命令里。注意后面的分号，这个是结束find命令中-exec选项的命令序列，在实际使用的时候，要转义一下以免被shell理解错误。
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					[]
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					<p style="margin-top:0px;margin-bottom:10px;">
						中括号（brackets）。
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						1. 测试的表示，Shell会测试在[]内的表达式，需要注意的是，[]是Shell内建的测试的一部分，而非使用外部命令/usr/bin/test的链接；
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						2. 在数组的上下文中，表示数组元素，方括号内填上数组元素的位置就能获得对应位置的内容，如：
					</p>
                     <p style="margin-top:0px;margin-bottom:10px;">
                         3. 表示字符集的范围，在正表达式中，方括号表示该位置可以匹配的字符集范围。
				     </p>
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					[[]]
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					双中括号(double brackets)。这个结构也是测试，测试[[]]之中的表达式(Shell的关键字)。这个比单中括号更能防止脚本里面的逻辑错误，比如：&amp;&amp;,||,&lt;,&gt;操作符能在一个[[]]里面测试通过，但是在[]却不能通过。[[]]里面没有文件名扩展(filename expansion）或是词分隔符(Word splitting)，但是可以用参数扩展(Parameter expansion)和命令替换(command substitution)。不用文件名通配符和像空白这样的分隔符。注意，这里面如果出现了八进制，十六进制等，shell会自动执行转换比较。
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					$[...]
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					词表达表示整数扩展(integer expansion)，在方括号里面执行整数表达式。例：
                </td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					(())
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					双括号(double parentheses)。 表示整数扩展（integer expansion）。功能和上面的$[]差不多，但是需要注意的是，$[]是会返回里面表达式的值的，而(())只是执行，并不会返回值。两者执行后如果变量值发生变化，都会影响到后继代码的运行。可对变量赋值，可以对变量进行一目操作符操作，也可以是二目，三目操作符。
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					&gt;,&amp;&lt;,&gt;&amp;,&gt;&gt;,&lt;,&lt;&gt;
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					重定向(redirection)。
				<p>	重定向scriptname的输出到文件filename中去，如果文件存在则覆盖；，会重定向command的标准输出(stdout)和标准错误(stderr)到文件filename中；把command的标准输出(stdout)重定向到标准错误(stderr)中；，吧scriptname的输出（同&gt;)追加到文件filenmae中，如果文件不存在则创建。
打开filename这个文件用来读或者写，并且给文件指定i为它的文件描述符(file descriptor)，文件不存在就会创建。</p>
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					(command)&gt;,&lt;(command)
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					这是进程替换(Process Substitution)。
					<p>	使用的时候注意，括号和&lt;,&gt;之间是不能有空格的，否则报错。其作用有点类似通道，但和管道在用法上又有些不同，管道是作为子进程的方式来运行的，这个命令会在/dev/fd/下面产生类似/dev/fd/63,/dev/fd/62这类临时文件，用来传递数据。Mitchell个人猜测之所以用这种方法来传递，是因为前后两个不属于同一个进程，因此需要用共享文件的方式来传递资料(这么说其实管道也应该有同样的文件?)。网上有人说这个只是共享文件而已，但是经过测试，发现虽然有/dev/fd/63这样的文件产生，但是这个文件其实是指向pipe:[43434]这样的通道的链接。</p>
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					&lt;&lt;
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					双小于号(here-document[double less then marks])。这个也被称为Here-document，用来将后继的内容重定向到左侧命令的stdin中。&lt;&lt;可以节省格式化时间，别且使命令执行的处理更容易。在实作的时候只需要输入&lt;&lt;和终止标志符，而后（一般是回车后）你就可以输入任何内容，只要在最后的新行中输入终止标志符，即可完成数据的导入。使用here-document的时候，你可以保留空格，换行等。如果要让shell脚本更整洁一点，可以在&lt;&lt;和终止符之间放上一个连字符(-)。
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					&lt;&lt;&lt;
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					三个小于号(here-strings)。Here-字串和Here-document类似，here-strings语法：command [args] &lt;&lt;&lt;["]$word["]；$word会展开并作为command的stdin。
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					&lt;,&gt;
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					小于，大于号(ASCII Comparison)。ASCII比较，进行的是变量的ASCII比较，字串？数字?呃...这个...不就是ASCII比较么？
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					\&lt;...\&gt;
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					词界符(word boundary)。这个是用在正则表达式中的一个特殊分隔符，用来标记单词的分界。比如：the会匹配there，another，them等等，如果仅仅要匹配the，就可以使用这个词界符，\&lt;the\&gt;就只能匹配the了。
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					|
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					管道(pipe)。管道是Linux，Unix都有的概念，是非常基础，也是非常重要的一个概念。它的作用是将管道前（左边）的命令产生的输出(stdout)作为管道后（右边）的命令的输入(stdin)。如：ls | wc l，使用管道就可以将命令连接在一起。注意：管道是每一个进程的标准输出都会作为下一个命令的标准输入，期间的标准输出不能跨越管道作为后继命令的标准输入，如：
					<pre style="padding:9.5px;font-family:Monaco, Menlo, Consolas, 'Courier New', monospace;font-size:13px;color:#333333;border-top-left-radius:4px;border-top-right-radius:4px;border-bottom-right-radius:4px;border-bottom-left-radius:4px;margin-top:0px;margin-bottom:10px;word-break:break-all;white-space:pre-wrap;border:1px solid rgba(0, 0, 0, 0.14902);background-color:#F5F5F5;">cat filename | ls -al | sort
					##想想这个的输出? 同时，管道是以子进程来运行的，所以管道并不能引起变量改变。
					</pre>
               </td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					&gt;|
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					强制重定向(force redirection)。这会强制重写已经存在的文件。
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					&amp;
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					与号(Run job in background[ampersand])。如果命令后面跟上一个&amp;符号，这个命令将会在后台运行。有的时候，脚本中在一条在后台运行的命令可能会引起脚本挂起，等待输入，出现这种情况可以在原有的脚本后面使用wait命令来修复。
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					&amp;&amp;,||
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					逻辑操作符(logical operator)。在测试结构中，可以用这两个操作符来进行连接两个逻辑值。||是当测试条件有一个为真时返回0（真），全假为假；&amp;&amp;是当测试条件两个都为真时返回真(0)，有假为假。
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					-
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					减号，连字符(Hyphen/minus/dash)。 
					<p style="margin-top:0px;margin-bottom:10px;">
					1. 作为选项，前缀[option, prefix]使用。用于命令或者过滤器的选项标志；操作符的前缀。如：
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
					2. 用于stdin或者stdout的重定向的源或目的[dash].在tar没有bunzip2的程序补丁时，我们可以这样：
					注意：在实作的时候，如果文件名是以[-]开头的，那么在加上这个作为定向操作符的时候，可能会出错，此时应该为文件加上合适的前缀路径，以避免这种情况发生，同样的，在echo变量的时候，如果变量是以[-]开始，那么可能也会产生意想不到的结果，为了保险起见，可以使用双引号引用标量：
					</p>
                    <p style="margin-top:0px;margin-bottom:10px;">
						还有，这种表示方法不是Bash内建的，要达到此点的这种效果，需要看你使用的软件是否支持这种操作；
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						3. 表示先前的工作目录(previous working directory)，因此，如果你cd到其他目录下要放回前一个路径的时候，可以使用cd -来达到目的，其实，这里的[-]使用的是环境变量的$OLDPWD，注意：这里的[-]和前一点是不同的；
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						4. 减号或者负号，用在算术操作中。
					</p>
				</td>
			</tr>
			<tr>
<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					=
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					<p style="margin-top:0px;margin-bottom:10px;">
						等号(Equals)。
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						1. 赋值操作，给变量赋值，么有空格在等号两侧；
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						2. 在比较测试中作为比较符出现，这里要注意，如果在中括号中作为比较出现，需要有空格符在等号左右两侧。
					</p>
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					+
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					<p style="margin-top:0px;margin-bottom:10px;">
						加号(Plus)。
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						1. 算术操作符，表示加法；
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						2. 在正则表达式中，表示的是其前的这个匹配规则匹配最少一次;
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						3.在命令或过滤器中作为选项标记，在某些命令或者内置命令中使用+来启用某些选项，使用-来禁止；
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						4. 在参数替换(parameter substitution)中，+前缀表示替代值(当变量为空的时候，使用+后面的值)
					</p>
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					%
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					<p style="margin-top:0px;margin-bottom:10px;">
						百分号(modulo[percent sign])。
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						1.在算术运算中，这个是求模操作符，即两个数进行除法运算后的余数；
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						2. 在参数替换(parameter substitution)中，可以作为模式匹配。例子：
					</p>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					~
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					波浪号(Home directory[tilde])，这个和内部变量$HOME是一样的。默认表示当前用户的家目录（主目录），这个和~/效果一致，如果波浪号后面跟用户名，表示是该用户的家目录，
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					~+
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					当前的工作目录(current working directory)。这个和内置变量$PWD一样。
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					~-
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					前一个工作目录(previous working directory)。这个和内部变量$OLDPWD一致，之前的[-]也一样。
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					=~
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					Bash 版本3中有介绍，这个是正则表达式匹配。可用在[[]]测试中，比如：
<pre style="padding:9.5px;font-family:Monaco, Menlo, Consolas, 'Courier New', monospace;font-size:13px;color:#333333;border-top-left-radius:4px;border-top-right-radius:4px;border-bottom-right-radius:4px;border-bottom-left-radius:4px;margin-top:0px;margin-bottom:10px;word-break:break-all;white-space:pre-wrap;border:1px solid rgba(0, 0, 0, 0.14902);background-color:#F5F5F5;">var="this is a test message."
[[ "$var" =~ tf*message ]] &amp;&amp; echo "Sir. Found that." || echo "Sorry Sir. No match be found."
##你可以修改中间的正则表达式匹配项，正则表达式可以但不一定需要使用双引号括起来。
</pre>
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					^
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;">
					<p style="margin-top:0px;margin-bottom:10px;">
						脱字符(caret)。
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						1. 在正则表达式中，作为一行的行首(beginning-of-line)位置标志符；
					</p>
					<p style="margin-top:0px;margin-bottom:10px;">
						2. 在参数替换(Parameter substitution)中，这个用法有两种，一个脱字符(${var^})，或两个(${var^^})，分别表示第一个字母大写，全部大写的意思(Bash version &gt;=4)。
					</p>
				</td>
			</tr>
			<tr>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;border-bottom-left-radius:4px;">
					空白
				</td>
				<td style="padding:8px;line-height:20px;vertical-align:top;border-top-width:1px;border-top-style:solid;border-top-color:#DDDDDD;border-left-width:1px;border-left-style:solid;border-left-color:#DDDDDD;border-bottom-right-radius:4px;">
					空白符(Whitespace)。空白符不仅仅是指空格(spaces)，还包括制表符(tabs)，空行(blank lines)，或者这几种的组合。可用做函数的分隔符,分隔命令或变量，空行不会影响脚本的行为，因此可以用它来规划脚本代码，以增加可读性，在内置的特殊变量$IFS可以用来针对某些命令进行输入的参数进行分割，其默认就是空白符。在字符串或变量中如果有空白符，可以使用引号来规避可能的错误。
				</td>
			</tr>
		</tbody>
	</table>

怎样，你有多少是了解的呢？Mitchell在开始的Shell脚本时候，发现在这里面有好多都是不认识呢。

说明：

因为涉及到翻译，文中内容不一定完全翻译准确，如果你发现有错误的地方，还请包涵指正。

参考：

+ 本文主要内容来源：[Advanced Bash-Scripting Guide][Advanced Bash-Scripting Guide]  

+ 对话 [UNIX: !$#@*%  ][UNIX: !$#@*%  ]

+ [wikipedia的Here文档][wikipedia的Here文档]

参考内容为本篇成文之际给予[Mitchell](http://blog.useasp.net/)帮助较大的文章，在整个过程中还有很多网站信息给我提供了帮助，在此对他们的作者的无私贡献表示感谢！




{% highlight ruby %}
文档信息
--------------
* 版权声明：自由转载-非商用-非衍生-保持署名（创意共享3.0许可证）
* 转载: [Unix & Linux 编程语言](http://blog.useasp.net/category/21.aspx)
{% endhighlight %}

[Advanced Bash-Scripting Guide]: 	http://tldp.org/LDP/abs/html/index.html
[UNIX: !$#@*%  ]:		http://www.ibm.com/developerworks/cn/aix/library/au-spunix_clitricks/
[wikipedia的Here文档]:	http://zh.wikipedia.org/wiki/Here%E6%96%87%E6%A1%A3
[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
