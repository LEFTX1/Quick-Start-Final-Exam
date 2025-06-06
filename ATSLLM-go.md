resume_handler_test.go
1. 新文件上传成功 `TestHandleResumeUpload_Success_NewFile`
![[Pasted image 20250604221501.png]]

2. 重复文件上传 `TestHandleResumeUpload_DuplicateFile`
![[Pasted image 20250604221547.png]]



3. 原始简历消费者 `TestStartResumeUploadConsumer_ProcessMessageSuccess`
![[Pasted image 20250604221559.png]]


4. LLM 解析消费者 `TestStartLLMParsingConsumer_ProcessMessageSuccess`
![[Pasted image 20250604221618.png]]