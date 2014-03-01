# 流 (Streams)

Node makes extensive use of streams as a data transfer mechanism—for example, for reading and writing files and transmitting data over network sockets. Chapter 5 has already shown you the standard streams—stdin, stdout, and stderr. This chapter, which explores Node’s streams API in greater detail, presents the different types of streams, how they work, and their various applications. But before starting, you should be aware that the streams API, while an important part of the Node core, is listed as unstable in the official documentation.

## Working with Streams

## 可读流

### data Events
### The end Event
### The close Event


## 可写流
### write() 方法
### end() 方法
### drain 事件
### finish 事件
### close 和 error 事件
### The pipe() 方法
### 可写流示例

## 文件流
### createReadStream()
#### ReadStream流open事件

#### options 参数
#### WriteStream流 open 事件
#### bytesWritten 属性


## Compression Using the zlib Module
### Convenience Methods
## Summary