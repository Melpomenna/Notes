#GPU 
#D3D 

_______
#Cpp 
Сбор информации о рендере

```cpp

	void BeginFrame(ID3D12CommandQueue* commandQueue, UInt64 frameCounter)
	{
	
		m_frateCounter = frameCounter;
		FramgeData& frame = m_frame[m_frameCounter%NUM_FRAMES];
		commandQueue->GetClockCalibration((UINT64*)&frame.gpuSyncStamp,(UINT64*)&frame.cpuSyncStamp);
	}
	
	void EndFrame(ID3D12CommandQueue* commandQueue)
	{
	
		commandQueue->GetTimestampFrequency(&gpuFrequency);
		FrameData& oldFrame = m_frame[{m_frameCounter+1}%NUM_FRAMES];
		UInt64* resilvedStamps = nullptr;
		oldFrame.readbackBuffer->Map(0, nullptr, (void**)&resolvedStamps);
		{
			const Double gpuDeltaSecs = Double{resolveStamps[queryIndex]-oldFrame.gpuSyncStamp/Double{gpuFrequency}};
			TimeStamp cpuStamp = oldFrame.cpuSyncStamp+SecToTimeStamps(gpuDeltaSecs);
			switch(e.type)
			{
				case Event::EType::Enter: Profiling::EnterGPUZone{EGPUQueue::Graphics, e.literalName, cpuStamp}; break;
				case Event::EType::Leave: Profiling::LeaveGPUZone{EGPUQueue::Graphics, e.literalName, cpuStamp}; break;
			}
		}
		oldFrame.readbackBuffer->Unmap(0, nullptr);
	}


	void EnterScope(const Char* literalName, UInt32 commandListId, ID3D12GraphicsCommandList* apiCommandList)
	{
		FrameData& fram m_frames[m_frameCounter%NUM_FRAMES];
		apiCommandList->EndQuery(frame.queryHeap.Get(), D3D12_QUERY_TYPE_TIMESTAMP, UINT(queryIndex));
	}

	void ResolveEvents(UInt32 commandListId, ID3D12GraphicsCommandList* apiCommandList)
	{
		FrameData& frame = m_frames[m_frameCounter%NUM_FRAMES];
		apiCommandList->ResolveQueryData(frame.queryHeap.Get(), D3D12_QUERY_TYPE_TIMESTAMP, NUM_COMMAND_LIST_EVENTS*commandListId, UINT(cmdListData.count), frame.readbackBuffer.Get(), sizeof(UInt64)*NUM_COMMAND_LIST_EVENTS*commandListId);
	}
```

______