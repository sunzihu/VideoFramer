<template>
	<view class="container">
		<text class="title">视频帧提取器</text>
		
		<!-- 功能区 -->
		<view class="function-area">
			<!-- 拍摄照片区域 -->
			<view class="action-item" @tap="takePhoto">
				<view class="action-icon">
					<text class="icon-camera">📷</text>
				</view>
				<text class="action-label">拍照</text>
			</view>

			<!-- 拍摄视频区域 -->
			<view class="action-item" @tap="paisheVideo">
				<view class="action-icon">
					<text class="icon-video">🎥</text>
				</view>
				<text class="action-label">拍摄视频</text>
			</view>

			<!-- 上传视频区域 -->
			<view class="action-item" @tap="uploadVideo">
				<view class="action-icon">
					<text class="icon-upload">⬆️</text>
				</view>
				<text class="action-label">上传媒体</text>
			</view>
		</view>

		<!-- 操作反馈 -->
		<view v-if="mediaPath" class="media-info">
			<text>{{ actionType === 'photo' ? '已拍摄照片' : actionType === 'record' ? '已录制视频' : '已选择视频' }}</text>
			<text v-if="actionType !== 'photo'" class="duration">时长: {{ durationLen }}秒</text>
			<text v-if="savedToAlbum" class="save-success">已保存到相册</text>

			<!-- 视频预览 -->
			<view v-if="actionType !== 'photo'" class="preview-container">
				<video 
					:src="mediaPath" 
					controls 
					class="preview-video" 
					id="idvideo"
					@timeupdate="onVideoTimeUpdate"
					@play="onVideoPlay"
					@pause="onVideoPause"
					@ended="onVideoEnded"
				></video>
			</view>
		</view>

		<!-- 提取按钮 -->
		<button class="extract-btn" @tap="extractFrames" :disabled="isExtracting || !mediaPath || actionType === 'photo'">
			{{ isExtracting ? '提取中...' : '每1秒提取一帧' }}
		</button>
		
		<!-- 传递视频路径给renderjs -->
		<view :prop="videoPathForRender" :change:prop="renderModule.extractFrames"></view>

		<!-- 进度显示 -->
		<view v-if="isExtracting" class="progress-container">
			<progress :percent="extractProgress" stroke-width="4"></progress>
			<text class="progress-text">提取进度: {{ extractProgress }}%</text>
		</view>

		<!-- 结果展示区域 -->
		<view v-if="hasResults" class="results-container">
			<text class="subtitle">提取的帧 (每1秒1帧)</text>
			
			<!-- 保存所有帧按钮 -->
			<button class="save-frames-btn" @tap="saveAllFramesToAlbum">
				保存所有帧到相册
			</button>
			
			<view class="frames-grid">
				<view v-for="(frame, index) in frames" :key="index" class="frame-item">
					<image :src="frame.path" :alt="'第' + frame.second + '秒的帧'" class="frame-image"></image>
					<text class="frame-index">第{{ frame.second }}秒</text>
					<button class="save-frame-btn" @tap="saveFrameToAlbum(frame.path, frame.second)">保存</button>
				</view>
			</view>
		</view>
	</view>
</template>

<script module="renderModule" lang="renderjs">
// 全局变量，用于存储renderjs实例
var renderJsInstance = null;

export default {
	mounted() {
		// 保存实例引用
		renderJsInstance = this;
		console.log('renderjs: 模块已挂载');
	},
	methods: {
		async extractFrames(newVal) {
			if (!newVal) return;
			
			console.log('renderjs: 开始提取帧，视频路径:', newVal);
			
			// 通知主线程开始提取
			this.$ownerInstance.callMethod('onExtractStart');
			
			// 使用Blob URL而不是直接设置src
			try {
				// 通知主线程我们需要视频数据
				this.$ownerInstance.callMethod('requestVideoData', { path: newVal });
				
				// 等待主线程提供视频数据
				// 由于renderjs无法直接接收回调，我们需要在主线程中处理
				// 这里先返回，后续操作将在主线程中进行
				return;
			} catch (e) {
				console.error('renderjs: 请求视频数据失败:', e);
				this.$ownerInstance.callMethod('onExtractError', { error: '请求视频数据失败' });
			}
		},
		
		// 新方法：接收视频数据并处理
		processVideoData(videoData) {
			if (!videoData) {
				console.error('renderjs: 未收到视频数据');
				this.$ownerInstance.callMethod('onExtractError', { error: '未收到视频数据' });
				return;
			}
			
			console.log('renderjs: 收到视频数据，开始处理');
			
			// 创建视频元素
			const video = document.createElement('video');
			video.crossOrigin = "anonymous"; // 设置跨域
			video.muted = true;
			video.autoplay = false;
			video.controls = false;
			
			// 使用主线程提供的视频数据
			video.src = videoData;
			
			// 视频加载完成后开始处理
			video.onloadedmetadata = async () => {
				console.log('renderjs: 视频元数据加载完成，时长:', video.duration);
				const duration = Math.floor(video.duration);
				const canvas = document.createElement('canvas');
				const ctx = canvas.getContext('2d');
				
				// 设置canvas尺寸
				canvas.width = video.videoWidth || 640;
				canvas.height = video.videoHeight || 480;
				
				console.log('renderjs: 视频尺寸:', canvas.width, 'x', canvas.height);
				
				// 通知主线程总帧数
				this.$ownerInstance.callMethod('updateTotalFrames', { total: duration });
				
				// 创建一个数组来存储帧信息（不包含base64数据）
				const frameInfos = [];
				
				// 逐帧处理
				for (let sec = 1; sec <= duration; sec++) {
					// 通知主线程当前进度
					this.$ownerInstance.callMethod('updateProgress', { current: sec, total: duration });
					
					console.log('renderjs: 提取第', sec, '秒的帧');
					video.currentTime = sec;
					
					await new Promise(resolve => {
						video.onseeked = () => {
							try {
								// 清除画布
								ctx.clearRect(0, 0, canvas.width, canvas.height);
								
								// 绘制视频帧
								ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
								
								// 尝试获取帧数据
								try {
									// 将帧转换为base64
									const base64 = canvas.toDataURL('image/jpeg', 0.9);
									
									// 将帧信息添加到数组
									frameInfos.push({
										second: sec,
										index: sec - 1
									});
									
									// 单独发送每一帧的数据
									this.$ownerInstance.callMethod('receiveFrameData', { 
										second: sec, 
										index: sec - 1,
										data: base64 
									});
									
									console.log('renderjs: 第', sec, '秒帧提取成功');
								} catch (e) {
									console.error('renderjs: 帧转换失败:', e);
									// 通知主线程帧提取失败
									this.$ownerInstance.callMethod('onFrameError', { 
										second: sec,
										error: e.toString()
									});
								}
								resolve();
							} catch (e) {
								console.error('renderjs: 绘制帧错误:', e);
								resolve();
							}
						};
					});
				}
				
				// 通知主线程所有帧处理完成
				console.log('renderjs: 所有帧提取完成，共', frameInfos.length, '帧');
				this.$ownerInstance.callMethod('onExtractComplete', { frames: frameInfos });
			};
			
			// 处理视频加载错误
			video.onerror = (e) => {
				console.error('renderjs: 视频加载错误:', e);
				this.$ownerInstance.callMethod('onExtractError', { error: '视频加载失败' });
			};
		}
	}
}

// 暴露全局方法，用于主线程调用
window.processVideoDataFromMain = function(videoData) {
	console.log('renderjs: 通过全局方法接收到视频数据');
	if (renderJsInstance) {
		renderJsInstance.processVideoData(videoData);
	} else {
		console.error('renderjs: 实例未初始化');
	}
};
</script>

<script>
	export default {
		data() {
			return {
				mediaPath: null,
				videoPathForRender: '', // 专门用于传递给renderjs的路径
				durationLen: 0,
				actionType: '',
				savedToAlbum: false,
				frames: [], // 存储所有提取的帧
				hasResults: false,

				isExtracting: false,
				extractProgress: 0,
				totalFramesToExtract: 0,
				currentExtractIndex: 0
			};
		},
		methods: {
			// 拍摄照片
			takePhoto() {
				uni.chooseImage({
					count: 1,
					sourceType: ['camera'],
					sizeType: ['compressed'],
					success: (res) => {
						this.mediaPath = res.tempFilePaths[0];
						this.actionType = 'photo';
						this.savedToAlbum = false;
						this.frames = []; // 清空帧数据
						this.hasResults = false;
						uni.showToast({
							title: '拍照成功',
							icon: 'success'
						});
						this.saveImageToAlbum(res.tempFilePaths[0]);
					},
					fail: (err) => {
						console.error('拍照失败:', err);
						uni.showToast({
							title: '拍照失败',
							icon: 'none'
						});
					}
				});
			},
			
			// 保存图片到相册
			saveImageToAlbum(filePath) {
				uni.saveImageToPhotosAlbum({
					filePath: filePath,
					success: () => {
						this.savedToAlbum = true;
						uni.showToast({
							title: '图片已保存到相册',
							icon: 'success'
						});
					},
					fail: (err) => {
						console.error('保存图片失败:', err);
						uni.showToast({
							title: '保存失败',
							icon: 'none'
						});
					}
				});
			},
			
			// 拍摄视频
			paisheVideo() {
				uni.chooseVideo({
					compressed: true,
					sourceType: ['camera'],
					maxDuration: 1800,
					success: (res) => {
						this.handleVideoSelection(res.tempFilePath, res.duration, 'record');
						this.saveVideoToAlbum(res.tempFilePath);
					},
					fail: (err) => {
						uni.showToast({
							title: '拍摄失败',
							icon: 'none'
						});
					}
				});
			},
			
			// 保存视频到相册
			saveVideoToAlbum(filePath) {
				uni.saveVideoToPhotosAlbum({
					filePath: filePath,
					success: () => {
						this.savedToAlbum = true;
						uni.showToast({
							title: '视频已保存到相册',
							icon: 'success'
						});
					},
					fail: (err) => {
						console.error('保存视频失败:', err);
						uni.showToast({
							title: '保存视频失败',
							icon: 'none'
						});
					}
				});
			},
			
			// 上传视频
			uploadVideo() {
				uni.chooseVideo({
					success: (res) => {
						this.handleVideoSelection(res.tempFilePath, res.duration, 'upload');
					},
					fail: (err) => {
						uni.showToast({
							title: '选择视频失败',
							icon: 'none'
						});
					}
				});
			},
			
			// 处理视频选择结果
			handleVideoSelection(path, duration, type) {
				this.mediaPath = path;
				this.durationLen = Math.floor(duration);
				this.actionType = type;
				this.savedToAlbum = false;
				this.frames = []; // 清空帧数据
				this.hasResults = false;
				this.isExtracting = false;
				this.extractProgress = 0;
				
				// 获取视频信息
				uni.getVideoInfo({
					src: path,
					success: (res) => {
						console.log('视频信息:', res);
						this.durationLen = Math.floor(res.duration);
					},
					fail: (err) => {
						console.error('获取视频信息失败:', err);
					}
				});

				uni.showToast({
					title: '选择视频成功',
					icon: 'success'
				});
			},
			
			// 视频时间更新事件
			onVideoTimeUpdate(e) {
				// 视频播放时间更新
				if (!this.durationLen && e.detail && e.detail.duration) {
					this.durationLen = Math.floor(e.detail.duration);
					console.log(`视频时长: ${this.durationLen}秒`);
				}
			},
			
			// 视频播放事件
			onVideoPlay() {
				console.log('视频开始播放');
			},
			
			// 视频暂停事件
			onVideoPause() {
				console.log('视频已暂停');
			},
			
			// 视频播放结束事件
			onVideoEnded() {
				console.log('视频播放结束');
			},
			
			// 提取视频帧 - 使用renderjs
			extractFrames() {
				if (!this.mediaPath || this.actionType === 'photo') {
					uni.showToast({
						title: '请先选择视频',
						icon: 'none'
					});
					return;
				}
				
				// 检查视频时长是否大于等于1秒
				if (this.durationLen < 1) {
					uni.showToast({
						title: '视频时长不足1秒',
						icon: 'none'
					});
					return;
				}
				
				this.isExtracting = true;
				this.frames = []; // 清空原有结果
				this.hasResults = false;
				this.extractProgress = 0;
				
				// 显示提取进度提示
				uni.showLoading({
					title: '准备提取...'
				});
				
				// 将视频路径传递给renderjs
				console.log('将视频路径传递给renderjs:', this.mediaPath);
				this.videoPathForRender = this.mediaPath;
			},
			
			// renderjs回调：提取开始
			onExtractStart() {
				console.log('提取开始');
				uni.hideLoading();
			},
			
			// renderjs回调：请求视频数据
			requestVideoData(data) {
				console.log('renderjs请求视频数据:', data.path);
				
				// 使用uni-app的API读取视频文件
				plus.io.resolveLocalFileSystemURL(data.path, (entry) => {
					entry.file((file) => {
						// 创建FileReader读取文件
						const reader = new plus.io.FileReader();
						
						reader.onloadend = (e) => {
							// 获取视频数据
							const videoData = e.target.result;
							
							// 将视频数据传递给renderjs
							console.log('视频数据读取成功，传递给renderjs');
							
							// 使用setTimeout确保renderjs已准备好接收数据
							setTimeout(() => {
								// 使用全局方法调用renderjs的处理方法
								const webview = this.$mp.page.$getAppWebview();
								webview.evalJS('window.processVideoDataFromMain && window.processVideoDataFromMain("' + videoData.replace(/\\/g, '\\\\').replace(/"/g, '\\"') + '")');
							}, 500);
						};
						
						reader.onerror = (e) => {
							console.error('读取视频文件失败:', e);
							this.onExtractError({ error: '读取视频文件失败' });
						};
						
						// 以DataURL形式读取文件
						reader.readAsDataURL(file);
					}, (error) => {
						console.error('获取文件对象失败:', error);
						this.onExtractError({ error: '获取文件对象失败' });
					});
				}, (error) => {
					console.error('解析视频路径失败:', error);
					this.onExtractError({ error: '解析视频路径失败' });
				});
			},
			
			// renderjs回调：更新总帧数
			updateTotalFrames(data) {
				this.totalFramesToExtract = data.total;
				console.log('总帧数:', this.totalFramesToExtract);
			},
			
			// renderjs回调：更新进度
			updateProgress(data) {
				this.currentExtractIndex = data.current;
				this.extractProgress = Math.floor((data.current / data.total) * 100);
				console.log('提取进度:', this.extractProgress, '%');
			},
			
			// renderjs回调：接收单帧数据
			receiveFrameData(data) {
				console.log(`接收到第${data.second}秒的帧数据`);
				
				// 添加到帧数组
				this.frames.push({
					path: data.data,
					second: data.second
				});
			},
			
			// renderjs回调：提取完成
			onExtractComplete(data) {
				console.log('提取完成，共提取了', data.frames.length, '帧');
				this.hasResults = true;
				this.isExtracting = false;
				this.extractProgress = 100;
				
				uni.showToast({
					title: '提取完成',
					icon: 'success'
				});
			},
			
			// renderjs回调：单帧提取错误
			onFrameError(data) {
				console.error(`第${data.second}秒帧提取失败:`, data.error);
				// 不中断整个提取过程，只记录错误
			},
			
			// renderjs回调：提取错误
			onExtractError(data) {
				console.error('提取错误:', data.error);
				this.isExtracting = false;
				uni.hideLoading();
				uni.showToast({
					title: '提取失败: ' + data.error,
					icon: 'none'
				});
			},
			
			// 保存单个帧到相册
			saveFrameToAlbum(base64Path, second) {
				console.log(`尝试保存第${second}秒的帧`);
				
				try {
					// 确保base64格式正确
					let base64Data = base64Path;
					if (!base64Data.startsWith('data:image')) {
						base64Data = 'data:image/jpeg;base64,' + base64Data;
					}
					
					// 创建临时文件路径
					const filePath = `_doc/${Date.now()}_frame_${second}.jpg`;
					
					// 使用plus API保存
					const bitmap = new plus.nativeObj.Bitmap('frame_' + second);
					bitmap.loadBase64Data(base64Data, () => {
						bitmap.save(filePath, {}, () => {
							console.log('保存临时文件成功:', filePath);
							
							// 获取本地文件系统路径
							const localFilePath = plus.io.convertLocalFileSystemURL(filePath);
							console.log('本地文件路径:', localFilePath);
							
							// 保存到相册
							uni.saveImageToPhotosAlbum({
								filePath: localFilePath,
								success: () => {
									uni.showToast({
										title: `第${second}秒帧已保存`,
										icon: 'success'
									});
									bitmap.clear();
								},
								fail: (err) => {
									console.error('保存图片失败:', err);
									uni.showToast({
										title: '保存失败: ' + err.errMsg,
										icon: 'none'
									});
									bitmap.clear();
								}
							});
						}, (error) => {
							console.error('保存临时文件失败:', error);
							uni.showToast({
								title: '保存失败',
								icon: 'none'
							});
							bitmap.clear();
						});
					}, (error) => {
						console.error('加载Base64数据失败:', error);
						uni.showToast({
							title: '图片处理失败',
							icon: 'none'
						});
					});
				} catch (e) {
					console.error('保存图片异常:', e);
					uni.showToast({
						title: '保存失败: ' + e.message,
						icon: 'none'
					});
				}
			},
			
			// 保存所有帧到相册
			saveAllFramesToAlbum() {
				if (this.frames.length === 0) {
					uni.showToast({
						title: '没有可保存的帧',
						icon: 'none'
					});
					return;
				}
				
				uni.showLoading({
					title: '正在保存...'
				});
				
				let savedCount = 0;
				let failCount = 0;
				
				// 逐个保存帧
				const saveNextFrame = (index) => {
					if (index >= this.frames.length) {
						// 所有帧处理完成
						uni.hideLoading();
						if (failCount > 0) {
							uni.showToast({
								title: `已保存${savedCount}个帧，${failCount}个失败`,
								icon: 'none',
								duration: 2000
							});
						} else {
							uni.showToast({
								title: `已保存${savedCount}个帧`,
								icon: 'success'
							});
						}
						return;
					}
					
					const frame = this.frames[index];
					
					try {
						// 确保base64格式正确
						let base64Data = frame.path;
						if (!base64Data.startsWith('data:image')) {
							base64Data = 'data:image/jpeg;base64,' + base64Data;
						}
						
						// 创建临时文件路径
						const filePath = `_doc/${Date.now()}_frame_${frame.second}.jpg`;
						
						// 使用plus API保存
						const bitmap = new plus.nativeObj.Bitmap('frame_' + index);
						bitmap.loadBase64Data(base64Data, () => {
							bitmap.save(filePath, {}, () => {
								// 获取本地文件系统路径
								const localFilePath = plus.io.convertLocalFileSystemURL(filePath);
								
								// 保存到相册
								uni.saveImageToPhotosAlbum({
									filePath: localFilePath,
									success: () => {
										savedCount++;
										bitmap.clear();
										// 处理下一帧
										setTimeout(() => {
											saveNextFrame(index + 1);
										}, 200); // 添加延时，避免过快保存导致的问题
									},
									fail: (err) => {
										console.error(`保存第${frame.second}秒帧失败:`, err);
										failCount++;
										bitmap.clear();
										// 处理下一帧
										setTimeout(() => {
											saveNextFrame(index + 1);
										}, 200);
									}
								});
							}, (error) => {
								console.error('保存临时文件失败:', error);
								failCount++;
								bitmap.clear();
								// 处理下一帧
								setTimeout(() => {
									saveNextFrame(index + 1);
								}, 200);
							});
						}, (error) => {
							console.error('加载Base64数据失败:', error);
							failCount++;
							// 处理下一帧
							setTimeout(() => {
								saveNextFrame(index + 1);
							}, 200);
						});
					} catch (e) {
						console.error('保存图片异常:', e);
						failCount++;
						// 处理下一帧
						setTimeout(() => {
							saveNextFrame(index + 1);
						}, 200);
					}
				};
				
				// 开始保存第一帧
				saveNextFrame(0);
			}
		}
	};
</script>

<style>
	.container {
		padding: 20rpx;
	}
	
	.title {
		font-size: 36rpx;
		font-weight: bold;
		text-align: center;
		margin-bottom: 20rpx;
		display: block;
	}
	
	.subtitle {
		font-size: 32rpx;
		font-weight: bold;
		text-align: center;
		margin-bottom: 20rpx;
		display: block;
	}

	.function-area {
		display: flex;
		justify-content: space-around;
		padding: 30rpx 0;
		background-color: white;
		border-radius: 10rpx;
		margin-bottom: 20rpx;
	}

	.action-item {
		display: flex;
		flex-direction: column;
		align-items: center;
	}

	.action-icon {
		width: 80rpx;
		height: 80rpx;
		margin-bottom: 10rpx;
		display: flex;
		justify-content: center;
		align-items: center;
	}
	
	.icon-camera, .icon-video, .icon-upload {
		font-size: 50rpx;
	}

	.action-label {
		font-size: 28rpx;
		color: #333;
	}

	.media-info {
		padding: 20rpx;
		background-color: white;
		border-radius: 10rpx;
		margin-bottom: 20rpx;
	}

	.duration,
	.save-success {
		font-size: 26rpx;
		color: #666;
		margin-top: 10rpx;
		display: block;
	}

	.save-success {
		color: #09BB07;
	}

	.preview-container {
		margin-top: 20rpx;
	}

	.preview-video {
		width: 100%;
		height: 400rpx;
		background-color: #000;
		border-radius: 10rpx;
	}

	.extract-btn {
		width: 100%;
		padding: 20rpx 0;
		background-color: #007AFF;
		color: white;
		border: none;
		border-radius: 10rpx;
		font-size: 32rpx;
		margin-top: 20rpx;
	}

	.extract-btn:disabled {
		background-color: #CCCCCC;
	}

	.progress-container {
		margin-top: 20rpx;
		padding: 20rpx;
		background-color: white;
		border-radius: 10rpx;
	}

	.progress-text {
		text-align: center;
		font-size: 28rpx;
		margin-top: 10rpx;
		color: #666;
		display: block;
	}

	.results-container {
		margin-top: 20rpx;
		padding: 20rpx;
		background-color: white;
		border-radius: 10rpx;
	}

	.frames-grid {
		display: grid;
		grid-template-columns: repeat(3, 1fr);
		gap: 20rpx;
		margin-top: 20rpx;
	}

	.frame-item {
		border: 1rpx solid #eee;
		border-radius: 10rpx;
		overflow: hidden;
	}

	.frame-image {
		width: 100%;
		height: 200rpx;
		object-fit: cover;
	}

	.frame-index {
		padding: 10rpx;
		font-size: 24rpx;
		text-align: center;
		color: #666;
		display: block;
	}
	
	.save-frame-btn {
		font-size: 24rpx;
		padding: 6rpx 0;
		margin: 0 10rpx 10rpx 10rpx;
		background-color: #007AFF;
		color: white;
		border: none;
		border-radius: 6rpx;
	}
	
	.save-frames-btn {
		width: 60%;
		margin: 20rpx auto;
		padding: 15rpx 0;
		background-color: #09BB07;
		color: white;
		border: none;
		border-radius: 10rpx;
		font-size: 28rpx;
	}
</style>