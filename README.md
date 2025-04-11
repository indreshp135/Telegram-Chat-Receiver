[Fact]
        public async Task ResetCache_OnlyPausesAndResumesAppropriateJobs()
        {
            // Arrange
            var mockSchedulerFactory = new Mock<ISchedulerFactory>();
            var mockScheduler = new Mock<IScheduler>();
            
            mockSchedulerFactory.Setup(f => f.GetScheduler(It.IsAny<CancellationToken>()))
                .ReturnsAsync(mockScheduler.Object);
            
            // Set up test data
            var jobGroup1 = "Group1";
            var jobGroup2 = "Group2";
            var jobGroups = new List<string> { jobGroup1, jobGroup2 };
            
            var jobKey1 = new JobKey("Job1", jobGroup1);
            var jobKey2 = new JobKey("Job2", jobGroup1);
            var jobKey3 = new JobKey("Job3", jobGroup2); // This one will be already paused
            
            var jobKeys1 = new List<JobKey> { jobKey1, jobKey2 };
            var jobKeys2 = new List<JobKey> { jobKey3 };
            
            var trigger1 = new Mock<ITrigger>();
            trigger1.Setup(t => t.Key).Returns(new TriggerKey("Trigger1", jobGroup1));
            
            var trigger2 = new Mock<ITrigger>();
            trigger2.Setup(t => t.Key).Returns(new TriggerKey("Trigger2", jobGroup1));
            
            var trigger3 = new Mock<ITrigger>();
            trigger3.Setup(t => t.Key).Returns(new TriggerKey("Trigger3", jobGroup2));
            
            // Set up job groups and keys
            mockScheduler.Setup(s => s.GetJobGroupNames(It.IsAny<CancellationToken>()))
                .ReturnsAsync(jobGroups);
                
            mockScheduler.Setup(s => s.GetJobKeys(GroupMatcher<JobKey>.GroupEquals(jobGroup1), It.IsAny<CancellationToken>()))
                .ReturnsAsync(jobKeys1);
                
            mockScheduler.Setup(s => s.GetJobKeys(GroupMatcher<JobKey>.GroupEquals(jobGroup2), It.IsAny<CancellationToken>()))
                .ReturnsAsync(jobKeys2);
            
            // Set up triggers for jobs
            mockScheduler.Setup(s => s.GetTriggersOfJob(jobKey1, It.IsAny<CancellationToken>()))
                .ReturnsAsync(new List<ITrigger> { trigger1.Object });
                
            mockScheduler.Setup(s => s.GetTriggersOfJob(jobKey2, It.IsAny<CancellationToken>()))
                .ReturnsAsync(new List<ITrigger> { trigger2.Object });
                
            mockScheduler.Setup(s => s.GetTriggersOfJob(jobKey3, It.IsAny<CancellationToken>()))
                .ReturnsAsync(new List<ITrigger> { trigger3.Object });
            
            // Set up trigger states (Job3 is already paused)
            mockScheduler.Setup(s => s.GetTriggerState(trigger1.Object.Key, It.IsAny<CancellationToken>()))
                .ReturnsAsync(TriggerState.Normal);
                
            mockScheduler.Setup(s => s.GetTriggerState(trigger2.Object.Key, It.IsAny<CancellationToken>()))
                .ReturnsAsync(TriggerState.Normal);
                
            mockScheduler.Setup(s => s.GetTriggerState(trigger3.Object.Key, It.IsAny<CancellationToken>()))
                .ReturnsAsync(TriggerState.Paused);
            
            // Set up currently executing jobs
            var execJob1 = new Mock<IJobExecutionContext>();
            execJob1.Setup(j => j.FireInstanceId).Returns("instance1");
            var currentlyExecutingJobs = new List<IJobExecutionContext> { execJob1.Object };
            
            mockScheduler.Setup(s => s.GetCurrentlyExecutingJobs(It.IsAny<CancellationToken>()))
                .ReturnsAsync(currentlyExecutingJobs);
            
            // Set up ForteCacheSyncWorkflow job
            var syncJobKey = new JobKey(nameof(ForteCacheSyncWorkflow), "Forte");
            var jobDetail = new JobDetailImpl();
            jobDetail.Key = syncJobKey;
            jobDetail.JobDataMap = new JobDataMap();
            
            mockScheduler.Setup(s => s.GetJobDetail(syncJobKey, It.IsAny<CancellationToken>()))
                .ReturnsAsync(jobDetail);
            
            // Create the controller
            var controller = new ResetCacheController(mockSchedulerFactory.Object);
            
            // Act
            var result = await controller.ResetCache(CancellationToken.None);
            
            // Assert
            Assert.IsType<OkResult>(result);
            
            // Verify PauseAll was called (for safety)
            mockScheduler.Verify(s => s.PauseAll(It.IsAny<CancellationToken>()), Times.Once);
            
            // Verify that PauseJob was called for job1 and job2 (not already paused)
            mockScheduler.Verify(s => s.PauseJob(jobKey1, It.IsAny<CancellationToken>()), Times.Once);
            mockScheduler.Verify(s => s.PauseJob(jobKey2, It.IsAny<CancellationToken>()), Times.Once);
            
            // Verify that PauseJob was NOT called for job3 (already paused)
            mockScheduler.Verify(s => s.PauseJob(jobKey3, It.IsAny<CancellationToken>()), Times.Never);
            
            // Verify that interrupt was called for each executing job
            mockScheduler.Verify(s => s.Interrupt("instance1", It.IsAny<CancellationToken>()), Times.Once);
            
            // Verify that ResumeJob was called for job1 and job2 only (the ones we paused)
            mockScheduler.Verify(s => s.ResumeJob(jobKey1, It.IsAny<CancellationToken>()), Times.Once);
            mockScheduler.Verify(s => s.ResumeJob(jobKey2, It.IsAny<CancellationToken>()), Times.Once);
            
            // Verify that ResumeJob was NOT called for job3 (was already paused)
            mockScheduler.Verify(s => s.ResumeJob(jobKey3, It.IsAny<CancellationToken>()), Times.Never);
        }
