## �Ż�Ŀ��
ֻҪapp�����У��϶��ǻ����ĵ����ġ��������Ż���ʱ��Ӧ���ص��ע�쳣�ĺĵ���Ϊ��
�������ϲ�ѯ
�쳣�ĵ�һ���Ϊ
	1��CPU�쳣�����粻��ȷ��wakelockʹ�ã�
	2�������쳣��Ƶ������·���󣩡�
	3���������쳣������Ҫʹ��GPS�Ĺ��ܣ�ȴ������GPS��


## �������

### ��εó��ĵ���
һ��Ӧ������ʱ������ϵͳҲ�����У�������̨����Ҳ�����С���ôȥ�������ǵ�ǰӦ�õĺĵ����أ�

 Android ��4.1�汾����ϵͳ������battry infoģ�飬��¼һ��ʱ�������������Ĺ���״̬�Լ�ÿ��Ӧ�õĹ������顣
### ��κ���Ӧ�úĵ�
��ôȥ����һ��Ӧ�úĵ���٣��϶�������Ӧ��һ��ʱ�䣬Ȼ�����������ġ����ǣ�һ��Ӧ��ͨ���������ģ�飬���Ҫȫ�ֵļ���һ��Ӧ�úĵ磬����鷳��Ҳ��������ԡ����ԣ�����Ӧ����Ӧ�õ�ģ��Ϊ��λ����һ��ʱ���ڷ���ʹ�ø�ģ�飬Ȼ��ó��������ġ����磬Ҫ������Ƶ���ŵĵ������ġ����ǿ��Բ�����ƵһСʱ��Ȼ��ͨ��battry info�鿴�������ġ�  **�����������յĺ�����׼��Ӧ������ģ��Ϊ��׼���ֱ����**��
![�ĵ��¼��.png](https://upload-images.jianshu.io/upload_images/9243886-eb3ddf6deee8ae50.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## ����ʹ�ý���
��ȡϵͳ�ĵ�����android4.1�������˵����Ϣ����������ֻҪdump�����Ϳ����ˡ�
```adb shell dumpsys batterystats```
�ļ����������������������ͷ��
![image.png](https://upload-images.jianshu.io/upload_images/9243886-929731d730826c46.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

�Һã�google��Դ��һ�������������Battery Historian�����ӻ������Ϣ�� ����֮�����ǿ�������������

![image.png](https://upload-images.jianshu.io/upload_images/9243886-caf220b791fa9db8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### ʹ��Battery Historian
##### �������ݵ��
```
adb shell dumpsys batterystats --reset   ����豸����ʹ�ü�¼
adb shell dumpsys batterystats --enable full-wake-history  ���Ӧ�ó����wakelock��¼
������Ӧ�Ĳ���
adb shell dumpsys batterystats --disable full-wake-history �ر�Ӧ�ó����wakelock��¼
adb bugreport bugreport.zip ������¼
�Լ��Battery Historian ���� ����ʹ�� https://bathist.ef.lc/ �ϴ���¼��
```

## ����������Ƶ����ģ����кĵ��Ż���
1���������ʹ�ü�¼ adb shell dumpsys batterystats --reset
2�����Ӧ�ó����wakelock��¼ adb shell dumpsys batterystats --enable full-wake-history
3��������ʮ������Ƶ
4��adb shell dumpsys batterystats --disable full-wake-history  �ر�Ӧ�ó����wakelock��¼
5��adb bugreport bugreport.zip ����
6���ϴ� https://bathist.ef.lc/ ���з���

![](https://upload-images.jianshu.io/upload_images/9243886-51d60df59dd95a04.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/9243886-1b71f987d1e73ee6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/9243886-3d0120c915527993.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### ��������
��һ��ͼ ��ͼ�����ʽ���û�һ��ֱ�۵ĸ��ܡ����Կ���������ʲô�׶��½���죬�����������쳣�������

�ڶ�������ͼ����߿���ѡ��app���鿴app��ʹ����������Է��֣�����һ��Сʱ�
a�����ǲ�����Ƶ������17.60%�ĵ�����
b��cpuһֱ���ڻ���״̬
c��wifi��22�����ڴ�����236.68MB����
d��ʹ����4����ΪWindowManager��WakeLock
...�ȵ���Ϣ��

## GOOGLE���Ż�����
ʹ�ù��߿��԰������ǿ��ٶ�λ�쳣�ĺĵ����������ȴ�����׷���һЩ��̫���ԵĶ���ĵ�����������������һ�Σ�����û��ѹ���ȵȣ������ԣ����ǻ�Ӧ�ý�����飬����ʵ����������Ż���

�����Ż�������Google����������������

> There are three important things to keep in mind in keeping your app power-thrifty:
>1��Make your apps Lazy First.
2��Take advantage of platform features that can help manage your app's battery consumption.
3��Use tools that can help you identify battery-draining culprits.

���ǿ��Ը�����Щ�����������Ż���
### 1��Lazy First
����: ���ٶ���Ĳ������������ͨ�����������ټ���������ʺͼ��㡣ͨ���������������ٽӿڵ������������ͨ��ѹ�����������ݵĴ��䡣

�Ƴ�: �����������ô���������ǿ����Ƴ����񣬵ȴ��豸�ڳ���ʱ����ִ�С������ϴ��û����ݣ���̨���µ�����

�ϲ�:��������Ƿ���Ժϲ���һ��ִ�У������λ���cpu�����硣

��Ӧ�����ǵ���Ŀ��ֵ���Ż��ĵ㣺
#### a�����٣� 

������Ʒ����ץ�����֣����ǵ������ƺ�������û�о���ѹ���ġ� 
GZIP�ֽ��Ѿ���ΪInternet ��ʹ�÷ǳ��ձ��һ������ѹ����ʽ��һ��Դ��ı����ݿ�ѹ����ԭ��С��40��
![image.png](https://upload-images.jianshu.io/upload_images/9243886-73cd0f45d624a4fc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/9243886-d1ea42af1b0bf602.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

ĿǰApp�ں�server�·���ͼƬ��������png��ʽ�����Ƕ���JPEG��PNG��GIF�ȳ���ͼƬ��ʽ���Ż��Ѽ����ﵽ���£����Google��2010�������һ���µ�ͼƬѹ����ʽ �� WebP����ͼƬ���Ż��ṩ���µĿ��ܡ�
WebPΪ����ͼƬ�ṩ�����������ѹ��������ͬʱ������������֧��͸��ͨ�����ݹٷ�ʵ����ʾ������WebP���PNG����26%��С������WebP����ͬ��SSIM��Structural Similarity Index���ṹ�����ԣ������JPEG����25%~34%�Ĵ�С������WebPҲ֧��͸��ͨ������Сͨ��ԼΪ��ӦPNG��1/3��

��Ŀ�е���Ϣ֪ͨ�ǲ�����ѯʵ�ֵģ�ÿ��һ����ȥ�����������ȡ������Ϣ�� ���ǿ��Լ��ɵ��������͹��ߣ���ʵ����Ϣ���ͣ�����Ƶ���������硣

#### b���Ƴ�
��Բ�̫�������������ǿ����Ƴ�����ȵ���Ӧ���龰�ٴ��������磬Ӧ�õĸ��¡��������û����г�磬��������wifi��ʱ���Զ����и��¡� 
Android ��5.0��ʼ �ṩ��JobScheduler�����ǿ���ʹ�������ض������������״̬��wifi����״ִ̬������
����JobScheduler֧�ֵĴ���������

![image.png](https://upload-images.jianshu.io/upload_images/9243886-637b9d946f8f7e07.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



#### c���ϲ�
Can work be batched, instead of putting the device into an active state many times? For example, is it really necessary for several dozen apps to each turn on the radio at separate times to send their messages? Can the messages instead be transmitted during a single awakening of the radio?
�����Ҫ����Ժ�̨app�ģ�����ϵͳ�Ѿ������������Ż���

### 2��Take advantage of platform features that can help manage your app's battery consumption.
Android 5.0 Ϊ�����ṩ�� JobScheduler �������Ż�ϵͳ���У��ṩ��Դ�����ʣ���ʡ������
Android 6.0 ����Dozeģʽ��App Standbyģʽ   �� 
Dozeģʽͨ�����豸��ʱ�䴦������״̬ʱ�Ƴ�Ӧ�õĺ�̨CPU������Activity�����ٵ������
App Standby Ӧ���л�����̨������ʱ��û�����û�����ʱ��ϵͳ������App Standbyģʽ������CPU�����������ٵ�����ġ�
Android 9.0 ����App Standby Buckets  ϵͳ�����û�ʹ����Ϊʹ�û���ѧϰ�㷨����Ӧ�÷����ڲ�ͬ��Buckets���档ϵͳ����buckets������������Ӧ����Ϊ��

![image.png](https://upload-images.jianshu.io/upload_images/9243886-4d762b27623a15c6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**������Ҫע��** ����Ϊƽ̨�������ԣ����ܻᵼ��Ӧ�õĹ��ܳ����쳣����������Ӧ���е���Ϣ��ѯ����Ӧ�ý���App Standby  ģʽ����������ᱻ�Ƴٵ�������ĳ��ʱ��㷢������������![image.png](https://upload-images.jianshu.io/upload_images/9243886-d1494156f444aa81.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/9243886-fc94f7e42049df67.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




### 3��Use tools that can help you identify battery-draining culprits.
ʹ�ù��߷��� ?[Battery Historian](https://github.com/google/battery-historian)

## �ܽ�
�����Ż���ʵ��һ���ܴ�ĸ����ֿ����������ڴ��Ż���CPU�Ż��������Ż���UI�Ż����������Ż���

������˵�����ڵ����Ż���������������Ҫ����
1����ģ���������Ų��쳣�ĵ磬����쳣�ĵ硣
2��д����ʱҪ����ʶ��ȥ����UI���ڴ桢CPU������Ȳ������������õ�ϰ��






















