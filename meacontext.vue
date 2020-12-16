<template>
  <view class="content">
    <!-- 实现蓝牙功能区 -->
	<u-row>
		<u-col span=12>
			<u-button type="success" @click="openBluetoothAdapter">开始扫描(初始化蓝牙)</u-button><br />
		</u-col>
	</u-row>
    <u-row>
		<u-col span=12>
			<u-button type="warning" @click="stopBluetoothDevicesDiscovery" >停止扫描(停止蓝牙搜索)</u-button><br />
		</u-col>
	</u-row>
	<u-row>
		<u-col span=12>
			<u-button type="error" @click="closeBluetoothAdapter">结束流程(关闭蓝牙适配器)</u-button><br />
		</u-col>
	</u-row>
	<u-row>
		<u-col span=12>
			<u-button type="primary" :disabled='myTemplateKey.length==0' @click="sendStorage">发送数据</u-button><br />
		</u-col>
	</u-row>
	<!-- 搜索蓝牙设备的对话框 -->
    <view class="devices_summary" v-if="!connected">已发现 {{ devices.length }} 个外围设备：</view>
    <scroll-view  v-if="!connected" class="device_list" scroll-y scroll-with-animation>
      <view
        v-for="(item, index) in devices"
        :key="index"
        :data-device-id="item.deviceId"
        :data-name="item.name || item.localName"
        @click="createBLEConnection($event)"
        class="device_item"
        hover-class="device_item_hover"
		>
        <view style="font-size: 16px; color: #333">{{ item.name }}</view>
        <view style="font-size: 10px">信号强度: {{ item.RSSI }}dBm ({{ myMax(0, item.RSSI + 100)}}%)
		</view>
        <view style="font-size: 10px">UUID: {{ item.deviceId }}</view>
        <view style="font-size: 10px">Service数量: {{ myLen(item.advertisServiceUUIDs) }}</view>
      </view>
    </scroll-view>
	<!-- 接收蓝牙模块传入数据的对话框 -->
	<view class="devices_summary" v-if="connected">
		<u-row >
			<u-col span='3'>
				<view>
					<u-button  @click="saveGroup">保存本组</u-button>
				</view>
			</u-col>
			<u-col span='3'>
				<view>
					<u-button  @click="saveStorage">保存本次</u-button>
				</view>
			</u-col>
			<u-col span='3'>
				<view>
					<u-button  @click="clearLocalStorage">清空本地</u-button>
				</view>
			</u-col>
			<u-col span='3'>
				<view>
					<u-button  @click="myDataMeasure=[]">清空接收栏</u-button>
				</view>
			</u-col>
		</u-row>
		<u-row>
			<u-col span='3'>
				<view>接收数据:</view>
			</u-col>
		</u-row>
	</view>
	<scroll-view v-if="connected" class="device_list" scroll-y scroll-with-animation>
		<view v-for="item,index in myDataMeasure" :key="index">
			{{item}}
		</view>
	</scroll-view>
	<!-- 显示连接对话框 -->
    <view class="connected_info" v-if="connected">
      <view>
        <text>已连接到 {{ deviceName }}</text>
        <view class="operation">
		  <button size="mini" @click="hideShowInfo" >{{infoShowHideButton}}</button>
          <button v-if="canWrite" size="mini" @click="writeBLECharacteristicValue" >写数据</button>
          <button size="mini" @click="closeBLEConnection">断开连接</button>
        </view>
      </view>
      <view v-show="infoShowHide" v-for="(item, index) in chs" :key="index" style="font-size: 12px; margin-top: 10px">
        <view>特性UUID: {{ item.uuid }}</view>
        <view>接收特性值(当前): {{ item.value }}</view>
      </view>
	  <view v-show="infoShowHide" v-if="canWrite">
		<u-input type='number' v-model="sendData"></u-input>
	  </view>
    </view>
  </view>
</template>

<script>
import { fail } from 'assert';
export default {
  data() {
    return {
	  //设备id
	  deviceId:'',
	  //服务id，这里根据自己的设备写死
	  serviceId:'0000FFE0-8888-1000-8000-0088888888888',
	  //特征值id-----这里根据自己的设备写死
	  characteristicId:'0000FFE1-8888-1000-8000-88888888',
	  //设备名称
	  deviceName:'',
      //存放蓝牙设备数组
	  devices: [],
	  //查看是否连接
	  connected: false,
	  //存储当前连接状态，里面的value就是手机接收的值
	  chs: [],
	  //记录电脑向手机传递的数据
	  myDataMeasure:[],
	  //查询蓝牙适配器的开始时间
	  _discoveryStarted: false, 
	  //是否可写
	  canWrite:false,
	  //要发送的数据
	  sendData:'',
	  //下部信息条的显示与隐藏(默认显示)
	  infoShowHide:true,
	  //隐藏和打开按钮的字
	  infoShowHideButton:'隐藏',
	  //用户对设备设置的基本参数
	  myData:{},
	  //临时存储仪器传输的数据以及基本设置信息
	  myMeatureDataPhone:[],
	  //存放数据的key，等待网络好的时候通过key传递数据到服务器
	  myTemplateKey:[],
	  //记录存储数据的时间
	  timeNow:''
    };
  },
  created() {
	//进入后刷新
	//uni.startPullDownRefresh();
  },
  onPullDownRefresh() {
	setTimeout(() => {
		console.log('VUEX数据',this.$store.state.myDataGlobal);
		console.log('VUEX数据时间间隔',this.$store.state.myDataGlobal.myInterval);
		console.log('实际蓝牙数据',this.myData);
		console.log('实际蓝牙输入框数据',this.sendData);
		uni.showToast({
			title:'参数更新成功'
		});
		uni.stopPullDownRefresh();
	}, 1000);
  },
  watch:{
	  '$store.state.myDataGlobal':{
		  deep:true,
		  immediate:true,
		  handler(){
			  //监听vuex数据变化，一旦变化及时取数据
			  this.myData=this.$store.state.myDataGlobal;
			  //获取的第一个数据就是时间间隔，后两个数据就是获取的数量（目前是不能超过99组，到时候可以考虑设置时间间隔，分开发送这两个数据）
			  this.sendData=((Number(this.myData.myInterval))*10).toString()+this.myData.myNumber
		  }
	  }
  },
  mounted(){
	
  },
  methods: {
	//发送存储中数据
	sendStorage(){
		uni.showLoading({
			title:'数据发送中...'
		});
		uni.getStorage({
			key:'mykey',
			success:(res)=>{
				this.myTemplateKey=res.data
				console.log('这是来自手机内存中的key',this.myTemplateKey);
			},
			fail:()=>{
				console.log('key不存在，无法发送');
				this.myTemplateKey=[];
				return;
			}
		});
		//查询key
		console.log('数据存储的key：',this.myTemplateKey);
		for(let i=0;i<this.myTemplateKey.length;i++)
		{
			console.log(`第${i}个:`,i)
			//打印storage中的文件
			uni.getStorage({
				key: this.myTemplateKey[i],//前后两次存的文件也都能找得到
				success: (res)=> {
					console.log(`打印存储在storage中${i}的数据`,res.data);
				},
				fail:()=>{
					console.log('该数据不存在');
				}
			});
		}
		//隐藏显示内容
		setTimeout(() => {
			uni.hideLoading();	
			uni.showToast({
				title:'查询并发送成功'
			})
		}, 1000);
	},
	//为了去除重复的
    inArray(arr, key, val) {
      for (let i = 0; i < arr.length; i++) {
        if (arr[i][key] === val) {
          return i;
        }
      }
      return -1;
	},
	//清空本地存储
	clearLocalStorage(){
		uni.showModal({
			title:'警告',
			content:'确认删除本地数据?请确认数据已及时发送',
			success: function (res) {
				if (res.confirm) {
					uni.clearStorage();
					uni.showToast({
						title:'本地数据已清空'
					});
				} else if (res.cancel) {
					console.log('用户点击取消');
					return;
				}
			}
		});
		
	},
	//将本次接受的数据保存
	saveStorage(){
		//先向数组中添加基本的设置信息
		let date=new Date();
		this.timeNow=date.getFullYear()+'-'+date.getMonth()+'-'+date.getDate()+' '+date.getHours()+':'+date.getMinutes()+':'+date.getSeconds();
		this.myMeatureDataPhone.push(this.timeNow);
		this.myMeatureDataPhone.push(this.myData.myInterval);
		this.myMeatureDataPhone.push(this.myData.myHomeStation);
		this.myMeatureDataPhone.push(this.myData.myOppositeStation);
		//将接收数据的数组进行合并
		this.myMeatureDataPhone=this.myMeatureDataPhone.concat(this.myDataMeasure);
		//查看新数组
		console.log('设置参数及数据数组内容：',this.myMeatureDataPhone);
		//查看本次的key
		console.log('本次的key:',this.myTemplateKey);
		//存储到本地storage中，并将key存到数组中，便于将来将数据获取后传递到服务器上。
		uni.setStorage({
			key: this.timeNow,
			data: this.myMeatureDataPhone,
			success:()=> {
				this.myTemplateKey.push(this.timeNow);
				//提示数据保存成功
				uni.showToast({
					title:'数据保存成功'
				});
				//清空数组准备存下一组
				this.myMeatureDataPhone=[];
			},
			fail:()=>{
				//提示数据保存失败
				uni.showToast({
					title:'数据保存失败',
					icon:'none'
				});
			}
		});	
	},
	//将本组的数据进行保存
	saveGroup(){
		//将key存储起来
		uni.setStorage({
			key:'mykey',
			data:this.myTemplateKey,
			success:()=>{
				uni.showToast({
					title:'本组已保存'
				});
				console.log('key存储成功')
			}
		});
	},
    // ArrayBuffer转16进度字符串示例
    ab2hex(buffer) {
      var hexArr = Array.prototype.map.call(
        new Uint8Array(buffer),
        function (bit) {
          return ("00" + bit.toString(16)).slice(-2);
        }
      );
      return hexArr.join("");
	},
	//将16进制字符串转换为10进制
	hex2int(hex) {
		var len = hex.length, a = new Array(len), code;
		for (var i = 0; i < len; i++) {
			code = hex.charCodeAt(i);
			if (48<=code && code < 58) {
				code -= 48;
			} else {
				code = (code & 0xdf) - 65 + 10;
			}
			a[i] = code;
		}
		return a.reduce(function(acc, c) {
			acc = 16 * acc + c;
			return acc;
		}, 0);
	},
	//比大小
	myMax(n1, n2) {
	  return Math.max(n1, n2)
	},
	//比长度
	myLen(arr) {
		arr = arr || []
		return arr.length
	},
	//显示隐藏下部信息条
	hideShowInfo(){
		this.infoShowHide=!this.infoShowHide;
		if(this.infoShowHideButton=='隐藏')
		{	
			this.infoShowHideButton='打开'
		}
		else
		{
			this.infoShowHideButton='隐藏'
		}
	},
    //开启蓝牙适配器初始化蓝牙模块
    openBluetoothAdapter() {
	  //刷新蓝牙设备
	  this.devices=[]
      uni.openBluetoothAdapter({
        success: (res) => {
		  console.log("开启蓝牙适配器成功(openBluetoothAdapter success)", res);
		  this.startBluetoothDevicesDiscovery();
		  uni.showToast({
            title: "开始扫描设备",
            icon: "success",
          });
        },
        fail: (res) => {
          uni.showToast({
            title: "请开启蓝牙",
            icon: "none",
          });
          if (res.errCode === 10001) {
            uni.onBluetoothAdapterStateChange(function (res) {
			  //监听蓝牙适配器是否打开，若打开则自动搜索蓝牙设备(onBluetoothAdapterStateChange)
              if (res.available) {
                this.startBluetoothDevicesDiscovery();
              }
            });
          }
        },
      });
    },
    //开启蓝牙设备搜索
    startBluetoothDevicesDiscovery() {
      // 关闭蓝牙适配器的时候将其打开
      if (this._discoveryStarted) {
        return;
      }
      this._discoveryStarted = true;
      uni.startBluetoothDevicesDiscovery({
        allowDuplicatesKey: true, //允许重复上报同一个设备
        success: (res) => {
          console.log( "开始搜寻蓝牙设备成功(startBluetoothDevicesDiscovery success)", res);
          uni.showLoading({
            title: "正在搜索设备",
          });
		  this.onBluetoothDeviceFound();
        },
      });
    },
    //监听寻找到新设备的事件
    onBluetoothDeviceFound() {
      uni.onBluetoothDeviceFound((res)=>{
			if(res){
				uni.hideLoading();
			}
			res.devices.forEach(device=>{
			//过滤掉没有名字的设备
			if (!device.name && !device.localName) {
				return
			};	
			//这么操作是为了去除重复
			const foundDevices = this.devices//将数据中的数组复制一份,利用动态绑定方式，不断复制最新的数组
			const idx = this.inArray(foundDevices, 'deviceId', device.deviceId)
			if (idx === -1) {
				this.devices.push(device);//数组里没有的的就向里面添加数据，保证没有重复[uni写法]
			}
		})
		console.log(this.devices);
	  });
	},
	//【点击某个蓝牙设备时】将目前你点击的设备作为对象e传递进来
	createBLEConnection(e) {
		// console.log('createBLEConnection',e)
		const ds = e.currentTarget.dataset
		//将现在点击的设备id传递进去
		this.deviceId = ds.deviceId
		//将设备名称也传递给全局变量
		this.deviceName = ds.name
		uni.createBLEConnection({
			//这个deviceId不能删除，否则会报错【从这里往后要放一放】
			deviceId:this.deviceId,
			success: (res) => {
				this.connected=true,
				// console.log("设备已经连接(createBLEConnection)");
				console.log('连接时获取设备id',this.deviceId);
				setTimeout(() => {
					this.getBLEDeviceServices(this.deviceId);
				}, 1000);//这里必须要延迟才行这是一个bug
			},
			fail:(err)=>{
			  uni.showToast({
				title:'建立连接失败',
				icon:'none'
			  })
			  return
			}
		})
		this.stopBluetoothDevicesDiscovery()//此时停止蓝牙搜索
	},
	//获取蓝牙设备所有服务（走到这一步证明已经找到服务）
	getBLEDeviceServices(deviceId) {
		uni.getBLEDeviceServices({
			deviceId:deviceId,
			success: (res) => {
				// for (let i = 0; i < res.services.length; i++) {
				// 	if (res.services[i].isPrimary) {
				// 		//将服务id存储添加到全局服务id数组中
				// 		this.serviceId=res.services[i].uuid;
				// 		console.log(`服务id:${i}`,this.serviceId);
				// 		this.getBLEDeviceCharacteristics(deviceId, this.serviceId)//获取蓝牙设备某个服务中所有特征值	
				// 		return
				// 	}
				// }

				//serviceId固定，获取蓝牙设备某个服务中所有特征值【读写属性的】
				this.getBLEDeviceCharacteristics(deviceId, this.serviceId)
			},
			fail:(err)=>{
			  uni.showToast({
				  title:'获取服务失败',
				  icon:'none'
			  })
			  return
			}
	  })
	},
	//如果一个蓝牙设备需要进行数据的写入以及数据传输，就必须具有某些特征值，所以通过上面步骤获取的id可以查看当前蓝牙设备的特征值
	getBLEDeviceCharacteristics(deviceId, serviceId) {
		uni.getBLEDeviceCharacteristics({
		//设备id与服务id必须要给,特征值id先不用获取，直接写死
		deviceId,
		serviceId,
		success: (res) => {
			if(res.characteristics[0].properties.read)
			{
				console.log('该特征值可读');
				uni.readBLECharacteristicValue({
					deviceId,
					serviceId,
					characteristicId:this.characteristicId,
				});
			}
			if(res.characteristics[0].properties.write)
			{
				console.log('该特征值可写');
				this.canWrite=true;
				//调用写
				this.writeBLECharacteristicValue()
			}
			//确保对应服务id下的特征值id具备监听数据变化的特性
			if (res.characteristics[0].properties.notify || res.characteristics[2].properties.indicate) {
				uni.notifyBLECharacteristicValueChange({
					deviceId,
					serviceId,
					characteristicId: this.characteristicId,
					state: true,//是否启用notify通知
					success:(res)=>{
						console.log('通知启用(notifyBLECharacteristicValueChange)',res);
					}
				})
			}
		},
		fail(res) {
			console.error('获取蓝牙设备特征值失败(getBLEDeviceCharacteristics)', res)
		}
		})
		// 操作之前先监听，保证第一时间获取数据（value就是特征值当前最新的值）
		//监听低功耗蓝牙设备的特征值变化事件。必须先启用 notifyBLECharacteristicValueChange 接口才能接收到设备推送的 notification
		uni.onBLECharacteristicValueChange((characteristic) => {
			console.log('触发监听(onBLECharacteristicValueChange)');
			console.log('手机接收的数据：',this.ab2hex(characteristic.value));
			//记录手机接受的数据
			this.myDataMeasure.push(this.ab2hex(characteristic.value));
			//记录目前通信的对象（和谁通信，特征值是多少，初始值00）
			const idx = this.inArray(this.chs, 'uuid', characteristic.characteristicId)
			const data = {}
			if (idx === -1) {
				this.chs.push({
					uuid: characteristic.characteristicId,
					value: this.ab2hex(characteristic.value)
				})
			} else {
				this.chs[idx] = {
					uuid: characteristic.characteristicId,
					value: this.ab2hex(characteristic.value)
				}
			}
			//打印手机接收的数据的数组
			console.log(this.myDataMeasure);
		})
	},
	//向蓝牙设备发送一个数据（期间要进行数据转换）
	writeBLECharacteristicValue() {
		console.log('我正在发送数据【手机向蓝牙模块】');
		//向蓝牙设备发送一个0x00的16进制数据//代表内存之中的一段二进制数据
		let buffer = new ArrayBuffer(1)
		//可以自定义复合格式的视图
		let dataView = new DataView(buffer)
		dataView.setUint8(0, this.sendData)

		uni.writeBLECharacteristicValue({
			deviceId: this.deviceId,
			serviceId: this.serviceId,
			characteristicId: this.characteristicId,
			value: buffer,
			complete:()=>{
				//buffer本身是arraybuffer（arraybuffer要转换为16进制）
				console.log('十六进制',this.ab2hex(buffer));
				let sixNumber=this.ab2hex(buffer);
				console.log('十进制',this.hex2int(sixNumber));
			}
		})
		//this.sendData=''
	},
    //停止蓝牙设备搜寻
    stopBluetoothDevicesDiscovery() {
	  uni.stopBluetoothDevicesDiscovery();
	  console.log("已停止蓝牙扫描(closeBluetoothAdapter)");
	  uni.showToast({
		title: "蓝牙搜索已停止",
		icon: "success",
	  });
    },
    //关闭搜索
    closeBlueTooth() {
	  uni.stopBluetoothDevicesDiscovery();
	  console.log("已关闭蓝牙搜索(closeBlueTooth)");
	},
	//关闭蓝牙适配器
	closeBluetoothAdapter() {
	  uni.closeBluetoothAdapter();
	  this._discoveryStarted = false;
	  console.log("已关闭蓝牙适配器(closeBluetoothAdapter)");
	  //清空蓝牙设备列表
	  this.devices=[]
	  uni.showToast({
		title: "适配器关闭成功",
		icon: "success",
	  });
	},
	//关闭蓝牙连接
	closeBLEConnection() {
		uni.closeBLEConnection({
		deviceId: this.deviceId
		});
		this.connected=false;
		this.chs=[];
		this.canWrite=false;
		console.log("已关闭蓝牙连接(closeBLEConnection)");
		uni.showToast({
			title: "蓝牙连接已断开",
			icon: "success",
		});
	},
  },
};
</script>

<style scoped lang="scss">
page {
  color: #333;
}
.devices_summary {
  margin-top: 10px;
  padding: 10px;
  font-size: 16px;
}
.device_list {
  height:280px;
  margin: 50px 5px;
  margin-top: 0;
  border: 1px solid #eee;
  border-radius: 5px;
  width: auto;
}
.device_item {
  border-bottom: 1px solid #eee;
  padding: 10px;
  color: #666;
}
.device_item_hover {
  background-color: rgba(0, 0, 0, 0.1);
}
.connected_info {
  position: fixed;
  bottom: 0;
  width: 100%;
  background-color: #f0f0f0;
  padding: 10px;
  padding-bottom: 20px;
  margin-bottom: env(safe-area-inset-bottom);
  font-size: 14px;
  //min-height: 100px;
  box-shadow: 0px 0px 3px 0px;
}
.connected_info .operation {
  position: absolute;
  display: inline-block;
  right: 30px;
}
</style>
