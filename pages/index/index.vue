<template>
	<view class="content">
		<view class="text-area">
			<text class="title">温度：{{ temperature }} 电量:{{electricity}}</text>
		</view>
		<view class="text-area">
			<text class="title">设备名称:{{ deviceName }}</text>
		</view>
		<view><button type="primary" @click="getTemperature()">获取广播数据</button></view>
		<view>设置时间间隔步骤</view>
		<view><button type="primary" @click="setIntervalTime()">设置间隔</button></view>
		<view><button type="primary" @click="stop()">停止记录</button></view>
		<view><button type="primary" @click="getHistory()">获取历史记录</button></view>
		<view>常规操作</view>
		<view><button type="primary" @click="init()">初始化</button></view>
		<view><button type="primary" @click="closeBLEConnection()">断开连接</button></view>
		<view><button type="primary" @click="getCharacteristicData()">读取特征值数据</button></view>
	</view>
</template>

<script>
	export default {
		data() {
			return {
				//测试用
				isTest: false,
				//是否已经打开蓝牙，默认为false，当蓝牙适配器初始化成功后为true
				isOpenBle: false,
				// 蓝牙是否已经成功打开过一次
				openBluetoothCode: -1,
				//温度的值
				temperature: '',
				//电量
				electricity: '',
				//设备系列号
				deviceName: '',
				//是否有外置温度
				isExternal: false,
				//间隔时间
				intervalTime: '1',
				//时区
				timeZone: 0,
				//历史数据的长度
				historyDatalength: 0,
				//历史数据
				historyData: [],
				//主服务的UUID
				primaryUUID: 'FFF0',
				//设备列表
				devicesList: [{
					device: '初始设备'
				}],
				//设备Id
				deviceId: '',
				//服务Id
				serviceId: '',
				//特征值ID
				characteristicId: {
					//cfg
					intervalId: '',
					//rtc
					resetTimeId: '',
					//历史数据
					historyDataId: '',
					//设置历史数据类型
					historyTypeId: ''
				},
				//控制命令
				wirteControlCode: {
					initRtc: '00000000000000'
				},
				//纪录状态
				status: {
					//是否已经写入cfg
					isWriteCfgSuccess: false,
					//是否已经写入rtc
					isWriteRTCSuccess: false,
					//是否已经读取历史数据长度
					isReadHistoryDataLength: false
				}
			};
		},
		onLoad() {
			this.init();
			let self = this;
			let count = 0;

			//监听蓝牙打开状态（若点击拒绝不会回调这里）
			uni.onBluetoothAdapterStateChange(function(res) {
				console.log(res);
				// 只有蓝牙曾经成功打开过才会回调
				count += 1;
				if (count == 2) {
					count = 0
					if (res.available) {
						//蓝牙打开成功
						self.init()
					} else {
						//蓝牙打开失败

						uni.closeBluetoothAdapter({
							//关闭蓝牙适配器
							success(res) {
								console.log(res);
							}
						});
						self.openBluetooth();
					}

				}
			})
		},
		onShow() {
			// 若是点击拒绝或者弹框之外  则会走这儿
			if (this.openBluetoothCode == 0) {
				this.openBluetoothCode = -1;
				uni.getBluetoothAdapterState({
					success: res => {
						if (!res.available) {
							console.log("蓝牙尚未打开");
							this.isOpenBle = false;
							this.openBluetooth();
						}
					},
					fail: res => {
						if (res.errCode == 10000) {
							console.log("蓝牙尚未打开");
							this.isOpenBle = false;
							this.openBluetooth();
						}
					}
				});
			}
		},
		methods: {
			//初始化蓝牙适配器
			init() {
				uni.openBluetoothAdapter({
					success: e => {
						console.log('初始化蓝牙成功:' + e.errMsg);
						if (!this.$data.isOpenBle) {
							this.$data.isOpenBle = true;
							//同时监听蓝牙连接状态
							this.onBLEConnectionStateChange();
							//读取特征值数据的回调（包括notify）
							this.onBLECharacteristicValueChange();
						}
					},
					fail: e => {
						console.log('初始化蓝牙失败，错误码：' + (e.errCode || e.errMsg));
						if (10001 == e.errCode) {
							this.openBluetooth();
						}
					}
				});
			},
			openBluetooth() {
				switch (uni.getSystemInfoSync().platform) {
					case 'android':
						let adapter = plus.android.importClass('android.bluetooth.BluetoothAdapter').getDefaultAdapter();
						if (!adapter.isEnabled()) {
							adapter.enable();
							this.openBluetoothCode = 0;
						}
						break;
					default:
						break;
				}
			},
			//获取广播数据（实时温度数据）
			getTemperature() {
				this.getTemperatureData(1000);
			},
			//设置时间间隔
			setIntervalTime() {
				//初始化
				//是否写入cfg成功
				this.status.isWriteCfgSuccess = false;
				//是否写入rtc成功
				this.status.isWriteRTCSuccess = false;

				let self = this;
				this.createBLEConnection(function() {
					//读取cfg
					// 读取之前先修改一下intervalTime【时间间隔】的值
					self.readBLECharacteristicValue(self.characteristicId.intervalId);
				});
			},
			//设置时间间隔成功【在处理后续业务逻辑】
			setIntervalTimeSuccess() {
				console.log("设置时间间隔成功");
			},
			//设置时间间隔失败【在处理后续业务逻辑】
			setIntervalTimeFail() {
				console.log("设置时间间隔失败，请稍后再试");
			},
			//停止记录
			stop() {
				let self = this;
				this.createBLEConnection(function() {
					self.writeBLECharacteristicValue("00000000000001", self.characteristicId.resetTimeId)
				})
			},
			//停止记录成功
			setStopSuccess() {
				console.log("停止记录成功")
			},
			//停止记录失败
			setStopFail() {
				console.log("停止记录失败")
			},
			getHistory() {
				//初始化
				this.status.isReadHistoryDataLength = false;
				this.historyData.length = 0;
				let self = this;
				this.createBLEConnection(function() {
					// 获取历史消息
					self.notifyBLECharacteristicValue();

					setTimeout(function() {
						if (self.isExternal) {
							self.writeBLECharacteristicValue('05', self.characteristicId.historyTypeId);
						} else {
							self.writeBLECharacteristicValue('02', self.characteristicId.historyTypeId);
						}
					}, 1000);
				})
			},
			//获取历史数据成功【在处理后续业务逻辑】
			getHistorySuccess() {
				console.log(this.historyData + '-------' + this.historyData.length);
			},
			//获取历史数据失败【在处理后续业务逻辑】
			getHistoryFail() {
				console.log("获取历史数据失败，请稍后再试");
			},
			getHistoryDataLength(hexArr) {
				// 获取历史消息长度
				let str = hexArr[1] + hexArr[0];
				return parseInt(str, 16);
			},
			getHistoryData(hexArr) {
				// 解析历史消息
				for (var i = 0; i < hexArr.length; i += 2) {
					let str = hexArr[i + 1] + hexArr[i];
					if (str !== "0000" && str !== "ffff") {
						this.historyData.push(parseInt(str, 16));
					}
				}
				if (this.historyData.length == this.historyDatalength) {
					return true;
				} else {
					return false;
				}
			},
			getCharacteristicData() {
				// 获取特征值数据
				this.isTest = true;
				this.readBLECharacteristicValue(this.characteristicId.resetTimeId);
			},
			//read或notify特征值数据的回调
			onBLECharacteristicValueChange() {
				let self = this;
				uni.onBLECharacteristicValueChange(function(res) {
					var resultArr = self.ab2hex(res.value);
					console.log(`特征值： ${res.characteristicId} 改变了---- ${resultArr}`);
					if (self.isTest) {
						return;
					}
					if (res.characteristicId.indexOf('FFFC') != -1) {
						//cfg
						if (!self.status.isWriteCfgSuccess) {
							//1.设置cfg

							//已经设置的时间间隔（单位：min）
							let intervalTime = parseInt(resultArr[1] + resultArr[0], 16);
							//获取设置cfg的命令
							let intervalTimeCode = self.getIntervalTimeWriteCode(self.intervalTime, resultArr);
							self.writeBLECharacteristicValue(intervalTimeCode, self.characteristicId.intervalId);
						}
					} else if (res.characteristicId.indexOf('FFF3') != -1) {
						//rtc
						
						if (!self.status.isWriteRTCSuccess) {
						
						} else {
						
						}
					} else if (res.characteristicId.indexOf('FFF4') != -1) {
						//历史数据
						if (self.status.isReadHistoryDataLength) {
							//如果已经读取过长度则写入指令去读取历史数据
							//是否已经读取结束
							let isover = self.getHistoryData(resultArr);
							if (isover) {
								self.getHistorySuccess();
							}
						} else {
							self.status.isReadHistoryDataLength = true;
							self.historyDatalength = self.getHistoryDataLength(resultArr);
							console.log("--------历史数据的长度：" + self.historyDatalength)
							// 写入指令去读取历史数据
							if (self.isExternal) {
								self.writeBLECharacteristicValue('04', self.characteristicId.historyTypeId);
							} else {
								self.writeBLECharacteristicValue('01', self.characteristicId.historyTypeId);
							}
						}

					}
				});
			},
			/**
			 *获取实时的温度
			 * duration  温度值更新的间隔
			 */
			getTemperatureData(duration) {
				//在页面显示的时候判断是都已经初始化完成蓝牙适配器若成功，则开始查找设备
				let self = this;
				setTimeout(function() {
					if (self.isOpenBle) {
						console.log('开始搜寻智能设备');
						uni.startBluetoothDevicesDiscovery({
							// services: ['0000fff0-0000-1000-8000-00805f9b34fb'],
							// allowDuplicatesKey:true,
							success: res => {
								self.onBluetoothDeviceFound();
								// self.getBluetoothDevices();
							},
							fail: res => {
								console.log('查找设备失败!');
							}
						});
					} else {
						console.log('未初始化蓝牙是配饰器：' + self.isOpenBle);
					}
				}, duration);
			},
			/**
			 * 发现外围设备
			 */
			onBluetoothDeviceFound() {
				uni.onBluetoothDeviceFound(devices => {
					this.getBluetoothDevices();
				});
			},
			/**
			 * 获取在蓝牙模块生效期间所有已发现的蓝牙设备。包括已经和本机处于连接状态的设备。
			 */
			getBluetoothDevices() {
				// console.log('获取蓝牙设备');
				uni.getBluetoothDevices({
					success: res => {
						this.devicesList = res.devices;
						//在这里查找对应名称的设备
						for (let i = 0; i < this.devicesList.length; i++) {
							let eq = this.devicesList[i];
							//Tps
							if (eq.name === 'Tps') {
								this.deviceId = eq.deviceId;
								let rssi = eq.RSSI;
								let arr = this.ab2hex(eq.advertisData);
								//处理设备系列号（唯一名称）
								this.deviceName = this.getDeviceName(arr);
								//处理电量
								this.electricity = parseInt(arr[19], 16);
								//【读取记录的状态】0没有启动记录  1待启动  2正在记录  3停止记录
								let readStatus = parseInt(arr[21], 16);
								//处理温度数据
								if (arr[17] === "ff" && arr[18] === "ff") {
									this.isExternal = false;
									//内置温度
									this.temperature = parseInt(arr[16] + arr[15], 16);
									console.log("没有安装外置接头");
								} else {
									this.isExternal = true;
									//外置温度
									this.temperature = parseInt(arr[18] + arr[17], 16);
								}
								console.log('设备系列号：' + this.deviceName +
									"\n电量:" + this.electricity +
									"\n读取记录的状态:" + readStatus +
									"\n温度（内或外）" + this.temperature
								);
								this.stopBluetoothDevicesDiscovery();
								// this.getTemperature();
								break;
							}
						}
					},
					fail: e => {
						console.log('获取蓝牙设备错误，错误码：' + e.errCode);
					}
				});
			},
			//获取设备唯一系列号
			getDeviceName(arr) {
				let result = "";
				for (var i = 0; i < 15; i++) {
					result += String.fromCharCode(parseInt(arr[i], 16));
				}
				return result;
			},
			/**
			 * 停止搜索蓝牙设备
			 */
			stopBluetoothDevicesDiscovery() {
				uni.stopBluetoothDevicesDiscovery({
					success: e => {
						console.log('停止搜索蓝牙设备:' + e.errMsg);
					},
					fail: e => {
						console.log('停止搜索蓝牙设备失败，错误码：' + e.errCode);
					}
				});
			},

			/**
			 * 连接设备
			 */
			createBLEConnection(fun) {
				if (!this.isOpenBle) {
					this.init();
				}
				console.log('开始连接...');
				//设备deviceId
				let deviceId = this.deviceId;
				let self = this;
				uni.createBLEConnection({
					// 这里的 deviceId 需要已经通过 createBLEConnection 与对应设备建立链接
					deviceId,
					success: res => {
						console.log('设备连接成功！');
						//延迟1.5s获取设备的services
						setTimeout(function() {
							self.getBLEDeviceServices(fun);
						}, 1500);
					},
					fail: res => {
						console.log(JSON.stringify(res));
						console.log('设备连接失败！');
					}
				});
			},
			/**
			 * 获取设备的服务ID
			 */
			getBLEDeviceServices(fun) {
				console.log('获取设备的services');
				let deviceId = this.deviceId;
				let serviceList = [];
				let self = this;
				uni.getBLEDeviceServices({
					// 这里的 deviceId 需要已经通过 createBLEConnection 与对应设备建立链接
					deviceId,
					success: res => {
						serviceList = res.services;
						for (let i = 0; i < serviceList.length; i++) {
							let service = serviceList[i];
							//比对service是否是FFF0服务
							if (service.uuid.indexOf(self.primaryUUID) != -1) {
								self.serviceId = service.uuid;
								console.log('设备的serviceId： ' + self.serviceId);
								//开始获取指定服务的特征值
								self.getBLEDeviceCharacteristics(fun);
								break;
							}
						}
					},
					fail: res => {
						console.log('device services:', res.errCode);
					}
				});
			},
			/**
			 * 获取指定服务的特征值
			 */
			getBLEDeviceCharacteristics(fun) {
				let deviceId = this.deviceId;
				let serviceId = this.serviceId;
				let characteristicsList = [];
				let self = this;
				uni.getBLEDeviceCharacteristics({
					// 这里的 deviceId 需要已经通过 createBLEConnection 与对应设备建立链接
					deviceId,
					// 这里的 serviceId 需要在 getBLEDeviceServices 接口中获取
					serviceId,
					success: res => {
						console.log('获取的' + serviceId + '服务的特征值：' + JSON.stringify(res.characteristics));
						characteristicsList = res.characteristics;
						for (let i = 0; i < characteristicsList.length; i++) {
							let characteristic = characteristicsList[i];
							// cfg
							if (characteristic.uuid.indexOf('FFFC') != -1) {
								self.characteristicId.intervalId = characteristic.uuid;
							} else if (characteristic.uuid.indexOf('FFF5') != -1) {
								// 判断获取历史数据相关的特征值
								self.characteristicId.historyTypeId = characteristic.uuid;
							} else if (characteristic.uuid.indexOf('FFF4') != -1) {
								// 获取历史数据相关的特征值
								self.characteristicId.historyDataId = characteristic.uuid;
							} else if (characteristic.uuid.indexOf('FFF3') != -1) {
								// rtc
								self.characteristicId.resetTimeId = characteristic.uuid;
							}
						}
						fun();
					},
					fail: res => {
						console.log('device getBLEDeviceCharacteristics failed:', JSON.stringify(res));
					}
				});
			},
			/**
			 * 获取设置时间间隔相关的指令
			 * */
			getIntervalTimeWriteCode(intervalTime, hexArr) {
				// console.log(0x100 + -12);
				// console.log((0x100 + -12).toString(16));
				// (value >= 0 ? value : 0x100000000 + value).toString(16) 
				//时区
				let time = 0 - new Date().getTimezoneOffset() / 60; // 输出：UTC+8
				this.timeZone = time;
				if (time >= 0) {
					hexArr[6] = ('0000' + time.toString(16)).slice(-2);
				} else {
					hexArr[6] = (0x100 + time).toString(16);
				}
				let tempArr = ('0000' + intervalTime.toString(16)).slice(-4).split('');
				hexArr[0] = tempArr[2] + tempArr[3];
				hexArr[1] = tempArr[1] + tempArr[0];
				//测试时暂写成01  新设备到了之后再改成00
				hexArr[2] = '00';
				hexArr[3] = '00';
				return hexArr.join('');
			},
			/**
			 * 获取设置时间的指令
			 * */
			getResetTimeWriteCode() {
				var date = new Date(),
					year = date.getFullYear(),
					month = date.getMonth() + 1,
					day = date.getDate(),
					hour = date.getHours() < 10 ? "0" + date.getHours() : date.getHours(),
					minute = date.getMinutes() < 10 ? "0" + date.getMinutes() : date.getMinutes(),
					second = date.getSeconds() < 10 ? "0" + date.getSeconds() : date.getSeconds();
				month >= 1 && month <= 9 ? (month = "0" + month) : "";
				day >= 0 && day <= 9 ? (day = "0" + day) : "";

				return ('00' + (year - 2000).toString(16)).slice(-2) +
					('00' + month.toString(16)).slice(-2) +
					('00' + day.toString(16)).slice(-2) +
					('00' + hour.toString(16)).slice(-2) +
					('00' + minute.toString(16)).slice(-2) +
					('00' + second.toString(16)).slice(-2) +
					"08";
			},

			// ArrayBuffer转16进度字符串示例
			ab2hex(buffer) {
				const hexArr = Array.prototype.map.call(new Uint8Array(buffer), function(bit) {
					return ('00' + bit.toString(16)).slice(-2);
				});
				return hexArr;
			},
			/**
			 * 开启订阅特征值
			 */
			notifyBLECharacteristicValue() {
				let deviceId = this.deviceId;
				let serviceId = this.serviceId;
				let characteristicId = this.characteristicId.historyDataId;
				let notify = true;
				let self = this;
				uni.notifyBLECharacteristicValueChange({
					state: true, // 启用 notify 功能
					// 这里的 deviceId 需要已经通过 createBLEConnection 与对应设备建立链接
					deviceId,
					// 这里的 serviceId 需要在 getBLEDeviceServices 接口中获取
					serviceId,
					// 这里的 characteristicId 需要在 getBLEDeviceCharacteristics 接口中获取
					characteristicId,
					success(res) {
						console.log('notifyBLECharacteristicValueChange success:' + JSON.stringify(res));
					},
					fail(res) {
						self.getHistoryFail();

					}
				});
			},
			/**
			 * 读取特征值数据
			 * characteristicId：特征值Id
			 */
			readBLECharacteristicValue(characteristicId) {
				let self = this;
				uni.readBLECharacteristicValue({
					// 这里的 deviceId 需要已经通过 createBLEConnection 与对应设备建立链接
					deviceId: this.deviceId,
					// 这里的 serviceId 需要在 getBLEDeviceServices 接口中获取
					serviceId: this.serviceId,
					// 这里的 characteristicId 需要在 getBLEDeviceCharacteristics 接口中获取
					characteristicId: characteristicId,
					success(res) {
						console.log('readBLECharacteristicValue:' + res.errMsg);
					},
					fail(res) {
						console.log('读取失败:' + res.errCode);
						this.setIntervalTimeFail();
					}
				});
			},
			/**
			 * 写入控制命令
			 * writeCode 写入的控制命令
			 */
			writeBLECharacteristicValue(writeCode, characteristicId) {
				let deviceId = this.deviceId;
				let serviceId = this.serviceId;
				let self = this;

				//因为协议文档中，一个字节两个字符的控制命令，codeLength为命令字节数
				let codeLength = writeCode.length / 2;
				const buffer = new ArrayBuffer(codeLength);
				const dataView = new DataView(buffer);

				//在这里解析将要写入的值
				for (let i = 0; i < codeLength; i++) {
					dataView.setUint8(i, '0X' + writeCode.substring(i * 2, i * 2 + 2));
					// console.log('次数：' + i + '-----' + writeCode.substring(2 * i, 2 * i + 2));
				}

				console.log('写入数据中deviceId：' + deviceId);
				console.log('写入数据中serviceId:' + serviceId);
				console.log('写入数据中characteristicId:' + characteristicId);
				console.log('发送的数据：' + writeCode);


				// console.log('发送的数据：');
				// for (let i = 0; i < dataView.byteLength; i++) {
				// 	console.log('0x' + dataView.getUint8(i).toString(16));
				// }

				uni.writeBLECharacteristicValue({
					// 这里的 deviceId 需要在 getBluetoothDevices 或 onBluetoothDeviceFound 接口中获取
					deviceId,
					// 这里的 serviceId 需要在 getBLEDeviceServices 接口中获取
					serviceId,
					// 这里的 characteristicId 需要在 getBLEDeviceCharacteristics 接口中获取
					characteristicId,
					// 这里的value是ArrayBuffer类型
					value: buffer,
					success(res) {
						console.log('写入数据成功:' + characteristicId, JSON.stringify(res));
						console.log('分割线************************************');
						if (characteristicId === self.characteristicId.intervalId) {
							//如果写入的是cfg
							self.status.isWriteCfgSuccess = true;
							//2.设置rtc reset_stop为0
							setTimeout(function() {
								self.writeBLECharacteristicValue(self.wirteControlCode.initRtc, self.characteristicId.resetTimeId);
							}, 500);
						} else if (characteristicId === self.characteristicId.resetTimeId) {
							if ("00000000000001" === writeCode) {
								self.setStopSuccess()
							}
							//已经写入rtc一次
							else if (!self.status.isWriteRTCSuccess) {
								self.status.isWriteRTCSuccess = true;
								//3.设置rtc reset_stop为08 并设置时间
								setTimeout(function() {
									let code = self.getResetTimeWriteCode();
									self.writeBLECharacteristicValue(code, self.characteristicId.resetTimeId);
								}, 500);
							} else {
								self.setIntervalTimeSuccess();
							}
						}
					},
					fail(res) {
						console.log('写入数据失败', res.errMsg);
						if ("00000000000001" === writeCode) {
							self.setStopFail()
						} else if (characteristicId === self.characteristicId.intervalId ||
							characteristicId === self.characteristicId.resetTimeId) {
							self.setIntervalTimeFail();
						} else if (characteristicId === self.characteristicId.historyTypeId) {
							self.getHistoryFail();
						}
					}
				});
			},
			/**
			 * 监听低功耗蓝牙连接状态的改变事件。包括开发者主动连接或断开连接，设备丢失，连接异常断开等等
			 */
			onBLEConnectionStateChange() {
				let count = 0;
				uni.onBLEConnectionStateChange(res => {
					// 该方法回调中可以用于处理连接意外断开等异常情况
					console.log(`蓝牙连接状态 -------------------------->`);
					console.log(JSON.stringify(res));
					if (!res.connected) {
						//在这里尝试重连
						//this.createBLEConnection();
						//关闭连接
						// this.closeBluetoothAdapter();
					}
				});
			},
			/**
			 * 断开蓝牙连接
			 */
			closeBLEConnection() {
				uni.closeBLEConnection({
					deviceId: this.deviceId,
					success: res => {
						console.log(res);
					}
				});
			}
		}
	};
</script>

<style>
	.content {
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
	}

	.logo {
		height: 200rpx;
		width: 200rpx;
		margin-top: 200rpx;
		margin-left: auto;
		margin-right: auto;
		margin-bottom: 50rpx;
	}

	.text-area {
		display: flex;
		justify-content: center;
	}

	.title {
		font-size: 36rpx;
		color: #8f8f94;
		width: 600rpx;
		height: 200rpx;
	}

	button {
		height: 100rpx;
		width: 600rpx;
		margin-top: 50rpx;
		margin-left: auto;
		margin-right: auto;
		margin-bottom: 50rpx;
	}
</style>
