---
title: golang学习
date: 2018-02-04
tags: 
---
 
 学习时把一些感想和思考记录在这里。
 
 先看的the tour of go, 然后go by example,
 
 channel是一种类型，连接goroutine,channel的类型由它传递的值的类型决定，如 chan string是传递string的channel，
 
 workspace, 工作空间，是一个目录，下面有src,pkg,bin三个目录
 src 包含go项目的代码
 pkg 包含包对象
 bin 包含可执行命令
 
 一个工作空间的示例：
 
		bin/
		    hello                          # command executable
		    outyet                         # command executable
		pkg/
		    linux_amd64/
		        github.com/golang/example/
		            stringutil.a           # package object
		src/
		    github.com/golang/example/
		        .git/                      # Git repository metadata
			hello/
			    hello.go               # command source
			outyet/
			    main.go                # command source
			    main_test.go           # test source
			stringutil/
			    reverse.go             # package source
			    reverse_test.go        # test source
		    golang.org/x/image/
		        .git/                      # Git repository metadata
			bmp/
			    reader.go              # package source
			    writer.go              # package source
		    ... (many more repositories and packages omitted) ...
		    	
　GOPATH 环境变量, 表示工作空间的路径，一般是home目录下的go目录。
　
　go run 运行程序
　go build 编译为二进制文件
		 
 
 