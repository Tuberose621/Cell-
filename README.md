**这里总结的是cell请求和返回数据许多细节方面的处理**
**一**
####自定义cell
  + 一般当系统不能满足我们的需求时，就要自定义
  + 发现cell固定，大小相同的时候勾选Xib，今后的cell从Xib中加载
  + cell一般要到缓存池中取，缓存池中没有就注册，重新创建
    + 注册时Xib中要设置Identifier为：tag
      + 勾选Xib的时候用registerNib
      + 没有的时候用registerClass
    + Xib是固定的，可以设置一个高度为70
- **有关于cell之间的分割线的设置**
- 系统的满足不了我们了要求太粗，我们需要自定义分割线，一般有两种方法
- 首先设置cell的的高，去掉系统自带的分割线

```objc

- (void)viewDidLoad {
    [super viewDidLoad];

    self.navigationItem.title = @"推荐标签";
    self.view.backgroundColor = CYCommonBgColor;
    // 设置行高
    self.tableView.rowHeight = 70;

    // 去掉系统自带的分割线
    self.tableView.separatorStyle = UITableViewCellSeparatorStyleNone;
    }
```

  + 然后再考虑怎么设置分割线
  + 方法一：添加一个UIView（设置它的frame：距左，右，下为0 。设定高为1，透明度为0.2）
    + 但是这种方法带来了许多的子控件，当我们的cell较多的时候不推荐这种方法
  + 方法二：保持cell的位置不变，高度都减1
    + 这样就可以少许多的子控件
    + 但是这样又有一些细节和注意的地方。分析如下：
- 有人会想在下面的cellForRowAtIndexPath:方法中修改cell的frame
  + 但是这是不可能的。因为这里是返回indexPath位置对应的cell，你如果是在这里修改后，一旦return cell。
  + 又会被TableView（系统）重新计算IndexPath对应cell的高度（前面你有有设定cell高度为70），frame等
  + 循环利用，又会被系统修改回去
  + 所以别想着在这里修改cell的frame

```objc
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
{
    CYTagCell *cell = [tableView dequeueReusableCellWithIdentifier:@"tag"];

    return cell;
}
```
- 也有人可能会想着实现代理方法，在下面的didSelectRowAtIndexPath:方法中修改cell的高度，每选中一次就让它的高度减1也可以实现分割线
  + 但是：同样的在上面的cellForRowAtIndexPath:方法中，用户一做上拉操作，就会循环利用，cell的高又会被系统重新修改回去


```objc
- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath
{
    CYTagCell *cell = (CYTagCell *)[tableView cellForRowAtIndexPath:indexPath];
    cell.height -= 1;
}
```
- 所以存在的问题：自己手动修改了cell的frame后，很有可能会被系统改回去
- 那么解决的方案：在系统设置完cell的frame后，自己再手动修改cell的frame, 重写setFrame:方法
  + 重写这个方法的目的：拦截cell的frame设置，系统改完后，我再改，永远改在系统的后面
  + 不用担心高度会递减，前面有设置cell的高度为70，在每次传进cell高度减1前，cell都会重新设置高度为原来的70

```objc
- (void)setFrame:(CGRect)frame
{
    frame.size.height -= 1;
    [super setFrame:frame];
}
```

- **重写setFrame:方法引申出来的价值**
- 1.当我们今后要拦截系统的某些设置（frame，size等），某些操作（像push）的时候。我们就要想到是不是可以去重写它们相应的方法
- 2.今后如果我们自定义的控件有一些特殊的需求，不希望外界可以轻松修改我们内部的值（像frame，size等）。或者是为了避免外界误传值修改了尺寸等值，我们就可以重写setFrame:方法
  + 当然别也可以通过修改bounds来修改尺寸，你就还得重写setBounds:方法，人家也还可以通过修改transform来修改，你就还得重写setTransform:方法。一般不会如此变态
  + 只是说记得有这么个细节和问题，知道可以这么干
  + 设置左右间距
    + x向右移动一个你定的间距margin，然后width减2倍间距margin
    + 就设置好了cell的分割线，以及左右间隔或者是左右间隔线

```objc
- (void)setFrame:(CGRect)frame
{
    frame.size.height -= 1;
    frame.origin.x = 5;
    frame.size.width -= 2 * frame.origin.x;

    [super setFrame:frame];
}
```

----------------------------
- **网络请求真实的数据**
- 首先，要注意的地方：
  + 使用Xcode7进行数据请求时
  + 苹果新特性要求App内访问网络请求，要采用 HTTPS 协议
  + 但是现在公司的项目使用的是 HTTP 协议，使用私有加密方式保证数据安全。现在也不能马上改成 HTTPS 协议传输。
    + 解决办法：
      + 在Info.plist中添加 NSAppTransportSecurity 类型 Dictionary
      + 在 NSAppTransportSecurity 下添加 NSAllowsArbitraryLoads 类型Boolean ,值设为 YES
        + 方式一: 使用文本编辑Info.plist, 在当中添加:

```html
<key>NSAppTransportSecurity</key>
<dict>
  <key>NSAllowsArbitraryLoads</key>
  <true/>
</dict>
```

+ 方式二: 在Info.plist中添加:

![](http://upload-images.jianshu.io/upload_images/739863-0b6f57fd41243d13.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


--------------------------------------
- 发送请求给服务器，获取数据
  + 加载标签数据
- 使用AFNetworking框架
  + 先要根据API文档请求参数
  + 参数是一个字典，里面有各种参数
- 发送请求

```objc
    // 加载标签数据
    // 请求参数
    NSMutableDictionary *params = [NSMutableDictionary dictionary];
    params[@"a"] = @"tag_recommend";
    params[@"action"] = @"sub";
    params[@"c"] = @"topic";

    // 发送请求
    [[AFHTTPSessionManager manager] GET:@"http://api.budejie.com/api/api_open.php" parameters:params success:^(NSURLSessionDataTask *task, id responseObject) {
        // 将服务器的数据写成plist。方便查看数据结构
        [responseObject writeToFile:@"/Users/gecongying/Desktop/tag.plist" atomically:YES];
    } failure:^(NSURLSessionDataTask *task, NSError *error) {

    }];
}
```
- 一般开发的适合你可以将服务器的数据写成plist。这样方便查看数据结构

![](http://upload-images.jianshu.io/upload_images/739863-d8916956b69ca97b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 可以看出plist文件中有许多的字典数组，所以我们要将字典转模型
- 字典转模型时，公司的服务器可能传回来一堆的数组，但是真正用到可以用于显示的数据只有几个，没必要每个都写，只取我需要的
  + 像这里只需要用到“头像”“订阅数”“name”三个数据
- 字典数组转模型数组。所以搞一个模型数组（新建标签模型CYTag）（方便开发），这里选用MJExtension框架

- 在CYTag.h文件中

```objc
#import <Foundation/Foundation.h>

@interface CYTag : NSObject
/** 图片 */
@property (nonatomic, copy) NSString *image_list;
/** 订阅数 */
@property (nonatomic, assign) NSInteger sub_number;
// 这里用NSInteger是为了后面更好的处理数字
/** 名字 */
@property (nonatomic, copy) NSString *theme_name;
@end
```
- 在CYTagViewController.m文件中
- 用一个标签数组保存标签数据（里面放的是CYTag模型）

```objc
#import "CYTagViewController.h"
#import "CYTagCell.h"
#import <AFNetworking.h>
#import <MJExtension.h>
#import "CYTag.h"

@interface CYTagViewController ()
/** 所有的标签数据（里面存放的都是CYTag模型） */
@property (nonatomic, strong) NSArray *tags;
@end

@implementation CYTagViewController
```
- 在TableView数据源方法中,cell返回的个数

```objc
#pragma mark - <UITableViewDataSource>
- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section
{
    return self.tags.count;
}
```

- 在CYTagCell.Xib文件中，将cell中的控件以及控件之间的束设置好。


![](http://upload-images.jianshu.io/upload_images/739863-0c5d9e3b6e42195f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 要把模型数组显示到cell对应的位置，所以得进行拖线,拖到CYTagcell.m文件中

```objc
@interface CYTagCell()
@property (weak, nonatomic) IBOutlet UIImageView *imageListView;
@property (weak, nonatomic) IBOutlet UILabel *themeNameLabel;
@property (weak, nonatomic) IBOutlet UILabel *subNumberLabel;
@end
```
- 字典转模型
- 字典转模型后要记得刷新表格

```objc
    // 发送请求
    [[AFHTTPSessionManager manager] GET:@"http://api.budejie.com/api/api_open.php" parameters:params success:^(NSURLSessionDataTask *task, id responseObject) {
        // 将服务器的数据写成plist。方便查看数据结构
        // [responseObject writeToFile:@"/Users/gecongying/Desktop/tag.plist" atomically:YES];

        // responseObject：字典数组
        // self.tags：模型数组
        // responseObject -> self.tags
        // 字典数组转模型数组
        self.tags = [CYTag objectArrayWithKeyValuesArray:responseObject];

        // 刷新表格
        [self.tableView reloadData];
    } failure:^(NSURLSessionDataTask *task, NSError *error) {

    }];
}
```


![](http://upload-images.jianshu.io/upload_images/739863-509e6e10f1868792.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

--------------------------------------
- CYTagCell内部要拿到相应的数据，所以也需要有标签模型
  - 下面的命名之所以用tagModel而不用tag
  - 因为系统控件基本都有tag属性，取名不正确会被系统覆盖
  - 所以有时候取名也得注意


```objc
@classCYTag;

@interface CYTagCell : UITableViewCell
/** 标签模型 */
@property (nonatomic, strong) CYTag *tagModel;
```

- setTagModel

```objc
- (void)setTagModel:(CYTag *)tagModel
{
    _tagModel = tagModel;
    self.themeNameLabel.text = tagModel.theme_name;
    self.subNumberLabel.text = tagModel.sub_number;
```

- 字典转模型，基本就以下三行代码：

```objc
/**
 * 返回indexPath位置对应的cell
 */
-(UITableViewCell *)tableView: (UITableView *)tableView cellForRowAtIndexPath:(nonnull NSIndexPath *)indexPath
{
    CYTagCell *cell = [tableView dequeueReusableCellWithIdentifier:@"tag"];
    cell.tagModel = self.tags[indexPath.row];
    return cell;
}
```

- 现在只剩下图片的下载
- 以及显示下载的图片，名字，以及订阅数
  + 在CYTagCell.m文件中用SDWebImage框架下载和显示图片
  + 显示标题
  + 订阅数-有关于数字的处理
    + 大于一万（tagModel.sub_number / 10000.0）与不到一万的处理
    + %.1f万人订阅--多少万人订阅，并保留1位小数

```objc
- (void)setTagModel:(CYTag *)tagModel
{
    _tagModel = tagModel;

    // 下载头像（SDWebImage框架给了一个占位图片defaultUserIcon）
    [self.imageListView sd_setImageWithURL:[NSURL URLWithString:tagModel.image_list] placeholderImage:[UIImage imageNamed:@"defaultUserIcon"]];

    self.themeNameLabel.text = tagModel.theme_name;
    // tagModel.sub_number = 678;
    // 订阅数
    // 有关于数字的处理
    if (tagModel.sub_number >= 10000) {
        self.subNumberLabel.text = [NSString stringWithFormat:@"%.1f万人订阅", tagModel.sub_number / 10000.0];
    } else {
        self.subNumberLabel.text = [NSString stringWithFormat:@"%zd人订阅", tagModel.sub_number];
    }
}
```
- 关于订阅数显示，可以加tagModel.sub_number = 678这句代码，用这样的假数据测试一下
- 显示结果：

![](http://upload-images.jianshu.io/upload_images/739863-fa1ed724b82fcd38.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

--------------------------------------

####细节处理部分

**用户网速慢的时候，网络有延迟的时候**
- 用户网速慢，网络有延迟的时候。会出现跳转后，没有内容，空白一片
- 下面模拟一下网速慢的时候出现的上述情况

```objc
    // 发送请求
    [[AFHTTPSessionManager manager] GET:@"http://api.budejie.com/api/api_open.php" parameters:params success:^(NSURLSessionDataTask *task, id responseObject) {
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(3.0 *NSEC_PER_SEC)), dispatch_get_main_queue(),^{
        self.tags = [CYTag objectArrayWithKeyValuesArray:responseObject];

        [self.tableView reloadData];
        });
```

![](http://upload-images.jianshu.io/upload_images/739863-c6b5c2a59db878d0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 用SVProgressHUD框架
  + 在数据加载请求前先弹一个框，数据请求成功后再隐藏掉
    + 这里是为了用户体验更好（网络慢，人家一进来就是一片空白什么鬼）
    + 也为了防止用户在数据请求前做操作或是直接点击离开界面（数据还没到呢）

```objc
[SVProgressHUD showWithMaskType:SVProgressHUDMaskTypeBlack];
```
- 数据回来后关闭弹框

```objc
[SVProgressHUD dismiss];
```

--------------------------------------
- **为了严谨期间，数据加载失败情况下，也要弹框**
  + 1.服务器出现问题，返回数据为空
  + 2.请求路径出错

```objc
# pragma mark- 专门加载标签数据
- (void)loadTags
{
    // 弹框
    //    [SVProgressHUD showWithMaskType:SVProgressHUDMaskTypeBlack];
    [SVProgressHUD show];

    // 请求参数
    NSMutableDictionary *params = [NSMutableDictionary dictionary];
    params[@"a"] = @"tag_recommend";
    params[@"action"] = @"sub";
    params[@"c"] = @"topic";

    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(3.0 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        // 发送请求
        [[AFHTTPSessionManager manager] GET:@"http://api.budejie.com/api/api_open.php" parameters:params success:^(NSURLSessionDataTask *task, id responseObject) {
            if (responseObject == nil) {
                // 关闭弹框
                [SVProgressHUD showErrorWithStatus:@"加载标签数据失败"];
                return;
            }

            self.tags = [CYTag objectArrayWithKeyValuesArray:responseObject];

            // 刷新表格
            [self.tableView reloadData];

            // 关闭弹框
            [SVProgressHUD dismiss];
        } failure:^(NSURLSessionDataTask *task, NSError *error) {
            // 关闭弹框
            [SVProgressHUD showErrorWithStatus:@"加载标签数据失败"];
        }];

    });

}
```

![](http://upload-images.jianshu.io/upload_images/739863-693d10a1aa5e7aa0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- **今后遇上此种网络请求问题，一定记得给用户弹框啊**

--------------------------------------
- 但是这样处理了还有隐患
  + 隐患一：假如数据请求不到的时候，你就会一直在那儿弹框
  + 隐患二：用户仍然可以点击上面的返回键，但是到了上一个界面，你还是在弹框

![](http://upload-images.jianshu.io/upload_images/739863-cfa944348e1a6ed0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  + 这又要怎么办呢？用户体验不行啊

- 方法一：在dealloc：方法中dismiss一下

```objc
- (void)dealloc
{
   [SVProgressHUD dismiss];
}
```
- 方法二：在控制器即将消失前dismiss掉(此法较为霸道)

```objc
- (void)viewWillDisappear:(BOOL)animated
{
    [super viewWillDisappear:animated];

    [SVProgressHUD dismiss];
}

```

--------------------------------------

- **在实际开发中遇上的问题**
  + 也是AFN框架自己的小问题
  + 用AFN框架的block时建议使用弱引用weakSelf


```objc
# pragma mark- 专门加载标签数据
- (void)loadTags
{
    // 弹框
    //    [SVProgressHUD showWithMaskType:SVProgressHUDMaskTypeBlack];
    [SVProgressHUD show];

    // 加载标签数据
    // 请求参数
    NSMutableDictionary *params = [NSMutableDictionary dictionary];
    params[@"a"] = @"tag_recommend";
    params[@"action"] = @"sub";
    params[@"c"] = @"topic";

    // 发送请求
    __weak typeof(self) weakSelf = self;
    [self.manager GET:@"http://api.budejie.com/api/api_open.php" parameters:params success:^(NSURLSessionDataTask *task, id responseObject) {
        if (responseObject == nil) {
            // 关闭弹框
            [SVProgressHUD showErrorWithStatus:@"加载标签数据失败"];
            return;
        }

        // responseObject：字典数组
        // weakSelf.tags：模型数组
        // responseObject -> weakSelf.tags
        weakSelf.tags = [CYTag objectArrayWithKeyValuesArray:responseObject];

        // 刷新表格
        [weakSelf.tableView reloadData];

        // 关闭弹框
        [SVProgressHUD dismiss];
    } failure:^(NSURLSessionDataTask *task, NSError *error) {
            // 关闭弹框
            [SVProgressHUD showErrorWithStatus:@"加载标签数据失败"];
        }
    }];
}

```

- **分析要用弱引用self的原因**
  + 因为用了self，成功success的block对我们self的控制器有一个强引用（这是因为manager有字典，它强引用着block。只要manager不死，block就不会死）这样的话控制器是不死的
  + 只有block死，self才会死，控制器也就才会死--这个时候没有block強指针指着它了
  + 而block只有请求完毕，调用完block后，那些代理方法删除后才会死（在didCompleteWithError：（请求完毕）方法中有一句：removeDelegateForTask）只有在这个时候，移除了一些代理方法后，block才会死
  + 那么控制器只有在服务器那儿请求完毕后才会销毁（也就是网络慢的时候的情况
    + 所以用弱引用self
    + 这样意味着，不管block死不死，我的控制器都不会被强引用着

--------------------------------------

- 网络请求仍有细节问题
  + 当发送请求时，网速慢，用户点击返回后，虽然控制器挂了，但是AFN内部仍然在请求网络数据，在浪费用户的流量
  + 所以要在控制器挂掉前，停止请求数据

- AFN请求数据主要由manager，所以要给manager设个属性，不管强弱（AFN内部早就封装存储好了，但最好是弱引用）
  + 一般框架内部的东西，我们最好用弱引用好些
  + 然后进行懒加载

```objc
/** 请求管理者 */
@property (nonatomic, weak) AFHTTPSessionManager *manager;
@end

@implementation CYTagViewController

- (AFHTTPSessionManager *)manager
{
    if (!_manager) {
        _manager = [AFHTTPSessionManager manager];
    }
    return _manager;
}
```
- 在dealloc：方法中，AFN内部有方法，让所有对象执行cancel方法，停止请求


![](http://upload-images.jianshu.io/upload_images/739863-347349c5dc8d8df4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




![](http://upload-images.jianshu.io/upload_images/739863-32c5d32cc296168e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 今后要管理AFN的所有任务，只要拿到manager就行了
-

![](http://upload-images.jianshu.io/upload_images/739863-14857c6a25d7c276.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- **细节问题**
- 判断是取消任务（code = -999）还是返回数据(域名，服务器，URL等)

![](http://upload-images.jianshu.io/upload_images/739863-c291b2953321ba52.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/739863-7c3577cd212bd76c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```objc
# pragma mark- 专门加载标签数据
- (void)loadTags
{
    // 弹框
    //    [SVProgressHUD showWithMaskType:SVProgressHUDMaskTypeBlack];
    [SVProgressHUD show];

    // 加载标签数据
    // 请求参数
    NSMutableDictionary *params = [NSMutableDictionary dictionary];
    params[@"a"] = @"tag_recommend";
    params[@"action"] = @"sub";
    params[@"c"] = @"topic";

    // 发送请求
    __weak typeof(self) weakSelf = self;
    [self.manager GET:@"http://api.budejie.com/api/api_open.php" parameters:params success:^(NSURLSessionDataTask *task, id responseObject) {
        if (responseObject == nil) {
            // 关闭弹框
            [SVProgressHUD showErrorWithStatus:@"加载标签数据失败"];
            return;
        }

        // responseObject：字典数组
        // weakSelf.tags：模型数组
        // responseObject -> weakSelf.tags
        weakSelf.tags = [CYTag objectArrayWithKeyValuesArray:responseObject];

        // 刷新表格
        [weakSelf.tableView reloadData];

        // 关闭弹框
        [SVProgressHUD dismiss];
    } failure:^(NSURLSessionDataTask *task, NSError *error) {
        // 如果是取消了任务，就不算请求失败，就直接返回
        if (error.code == NSURLErrorCancelled) return;

        if (error.code == NSURLErrorTimedOut) {
            // 关闭弹框
            [SVProgressHUD showErrorWithStatus:@"加载标签数据超时，请稍后再试！"];
        } else {
            // 关闭弹框
            [SVProgressHUD showErrorWithStatus:@"加载标签数据失败"];
        }
    }];
}


- (void)dealloc
{
    // 停止请求
    [self.manager invalidateSessionCancelingTasks:YES];

    //    [self.manager.downloadTasks makeObjectsPerformSelector:@selector(cancel)];
    //    [self.manager.dataTasks makeObjectsPerformSelector:@selector(cancel)];
    //    [self.manager.uploadTasks makeObjectsPerformSelector:@selector(cancel)];

    //    [self.manager.tasks makeObjectsPerformSelector:@selector(cancel)];

    //    for (NSURLSessionTask *task in self.manager.tasks) {
    //        [task cancel];
    //    }

    [SVProgressHUD dismiss];
}
```

--------------------------------------

- **网络请求几个细节总结**
  + 1.block里面不用self，用Weakself
  + 2.将manager存起来，方便管理任务Tasks请求
  + 3.有时候服务器返回的数据是空的，严谨起见，弹框提示并return（情况一般少见）
  + 4.在Failure这个block中做事情的时候，一定要分清楚情况：是取消任务还是别的原因来到Failure的
  + 5.控制器挂掉时，一定要将请求关闭，不要浪费用户的流量，并且弹框提示关闭

---------------------------
**设置圆角图片**

![](http://upload-images.jianshu.io/upload_images/739863-a0067211c9781e29.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 方法一
  + 用CALayer设置圆角图片，但是会出现卡屏现象
  + 苹果在渲染的时候做了许多耗时耗能的内部操作
  + 不推荐使用

```objc
- (void)awakeFromNib
{
    // 如果使用过于频繁，可能会导致拖拽起来的感觉比较卡
    self.imageListView.layer.cornerRadius = self.imageListView.width * 0.5;
    self.imageListView.layer.masksToBounds = YES;
}
```
- 方法二
  + 运用Quarz2D技术，将圆角图片画上去
  + if (image == nil) return;
    + 这句代码很有意义，如果没有返回数据，给一个占位图片

```objc
- (void)setTagModel:(CYTag *)tagModel
{
    _tagModel = tagModel;

    CYWeakSelf;
    // 下载头像（SDWebImage框架给力一个占位图片defaultUserIcon）
    [self.imageListView sd_setImageWithURL:[NSURL URLWithString:tagModel.image_list] placeholderImage:[UIImage imageNamed:@"defaultUserIcon"] completed:^(UIImage *image, NSError *error, SDImageCacheType cacheType, NSURL *imageURL) {
        // 如果图片下载失败，就不做任何处理，按照默认的做法：显示占位图片
        // 下面这句代码的意义是很大的
        if (image == nil) return;

        // 开启图形上下文
        UIGraphicsBeginImageContext(image.size);

        // 获得上下文
        CGContextRef ctx = UIGraphicsGetCurrentContext();

        // 矩形框
        CGRect rect = CGRectMake(0, 0, image.size.width, image.size.height);

        // 添加一个圆
        CGContextAddEllipseInRect(ctx, rect);

        // 裁剪(裁剪成刚才添加的图形形状)
        CGContextClip(ctx);

        // 往圆上面画一张图片
        [image drawInRect:rect];

        // 获得上下文中的图片
        self.imageListView.image = UIGraphicsGetImageFromCurrentImageContext();

        // 关闭图形上下文
        UIGraphicsEndImageContext();
    }];

    self.themeNameLabel.text = tagModel.theme_name;

    // 订阅数
    // 有关于数字的处理
    if (tagModel.sub_number >= 10000) {
        self.subNumberLabel.text = [NSString stringWithFormat:@"%.1f万人订阅", tagModel.sub_number / 10000.0];
    } else {
        self.subNumberLabel.text = [NSString stringWithFormat:@"%zd人订阅", tagModel.sub_number];
    }
}
```

- **封装一个UIImage的分类**
  + 在UIImage+CYExtension.h文件中

```objc
#import <UIKit/UIKit.h>

@interface UIImage (CYExtension)
/**
 * 返回一张圆形图片
 */
- (instancetype)circleImage;

/**
 * 返回一张圆形图片
 */
+ (instancetype)circleImageNamed:(NSString *)name;
@end
```
- + 在UIImage+CYExtension.m文件中

```objc
#import "UIImage+CYExtension.h"

@implementation UIImage (CYExtension)
- (instancetype)circleImage
{
    // 开启图形上下文
    UIGraphicsBeginImageContext(self.size);

    // 获得上下文
    CGContextRef ctx = UIGraphicsGetCurrentContext();

    // 矩形框
    CGRect rect = CGRectMake(0, 0, self.size.width, self.size.height);

    // 添加一个圆
    CGContextAddEllipseInRect(ctx, rect);

    // 裁剪(裁剪成刚才添加的图形形状)
    CGContextClip(ctx);

    // 往圆上面画一张图片
    [self drawInRect:rect];

    // 获得上下文中的图片
    UIImage *image = UIGraphicsGetImageFromCurrentImageContext();

    // 关闭图形上下文
    UIGraphicsEndImageContext();

    return image;
}

+ (instancetype)circleImageNamed:(NSString *)name
{
    return [[self imageNamed:name] circleImage];
}
@end

```
- **封装一个UIImageView设置头像的分类**
  + 在UIImageView+CYExtension.h文件中

```objc
#import <UIKit/UIKit.h>

@interface UIImageView (CYExtension)
/**
 * 设置头像
 */
- (void)setHeader:(NSString *)url;
@end
```
  + 在UIImageView+CYExtension.m文件中

```objc

#import "UIImageView+CYExtension.h"
#import <UIImageView+WebCache.h>

@implementation UIImageView (CYExtension)
- (void)setHeader:(NSString *)url
{
    [self setCircleHeader:url];
}

- (void)setRectHeader:(NSString *)url
{
    [self sd_setImageWithURL:[NSURL URLWithString:url] placeholderImage:[UIImage imageNamed:@"defaultUserIcon"]];
}

- (void)setCircleHeader:(NSString *)url
{
    CYWeakSelf;
    // PCH文件中有定义
    UIImage *placeholder = [[UIImage imageNamed:@"defaultUserIcon"] circleImage];
    [self sd_setImageWithURL:[NSURL URLWithString:url] placeholderImage:placeholder completed:
     ^(UIImage *image, NSError *error, SDImageCacheType cacheType, NSURL *imageURL) {
       // 如果图片下载失败，就不做任何处理，按照默认的做法：会显示占位图片
       if (image == nil) return;

       weakSelf.image = [image circleImage];
   }];
}
@end

```
- 今后不管你是要设置圆角还是直角的头像
- 只需要一行代码setCircleHeader:url或者是setRectHeader:url就行了

```objc
- (void)setHeader:(NSString *)url
{
    [self setCircleHeader:url]; // 只需要改这里了哦
}
```
- 例如：

```objc
- (void)viewDidLoad {
    [super viewDidLoad];

    UIImageView *imageView = [[UIImageView alloc] init];
    imageView.frame = CGRectMake(100, 100, 100, 100);
    [imageView setHeader:@"https://www.baidu.com/img/bd_logo1.png"];
    [self.view addSubview:imageView];
    }
```
------------------------------------------------------------
- 额外补充
- 1.cocoapods中的框架安装中platform :ios, '6.0'

```objc
platform :ios, '6.0'
pod 'AFNetworking'~> 0.8'
```
  - 这里写7.0，写8.0是有区别的
  - 这是表明你下载的AFNetworking框架只支持iOS6的
  - 如果有些功能只能在iOS7或者是iOS8中才有的，你是用不了的
  - 如果想用最新版的一些功能就改为高版本的iOS8.0

- 2.DEBUG中宏定义的问题
  + 这里可以创建一个宏，但是规定不能全是小写。如：age = 20；是不可以的
  + aGe = 20 / age8 = 20这样是可以的


![](http://upload-images.jianshu.io/upload_images/739863-5a61d962140f9656.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 3.有关于weak的问题
  +　一个对象死不死主要是看它有没有強指针指着

![](http://upload-images.jianshu.io/upload_images/739863-003696705b99523e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 4.IBOutlet的问题
  + IBOutlet内部有一个隐藏的强引用，它是由特殊用途的
  + 它的内部有强引用，为了在死之前做某些操作，具体在哪里不清楚，但是不会影响使用，我们能够知道这个现象就可以了。weak的指针是会消失的，但IBOutlet内部有一个强引用引用着，但是系统会在恰当的时候会将它去掉，不会影响weak的死亡时间。只是前后会有点差别，了解一下就好
  + 所以自定义 IBOutlet 控件属性一般也使用 weak

--------------------------------------

- **有关于全局变量使用**
- 默认情况下，全局变量是全世界都可以访问的
- 全局常量的写法
+ 1.仅限于本文件访问
  + 在本文件（.m）中写下面的代码

```objc
static 类型 const 常量名 = 常量值;
```
+ 2.全世界都要访问
   + 1> 在CYConst.m文件中

```objc
#import <UIKit/UIKit.h>
 类型 const 常量名 = 常量值;
```
```objc
// CYConst.m ：定义所有的全局常量

#import <UIKit/UIKit.h>

// 请求路径
NSString * const CYRequestURL = @"http://api.budejie.com/api/api_open.php";
```
   + 2> 在CYConst.h文件中

```objc
 #import <UIKit/UIKit.h>
 UIKIT_EXTERN 类型 const 常量名;
```
```objc
// CYConst.h ：引用所有的全局常量

#import <UIKit/UIKit.h>

// 请求路径
UIKIT_EXTERN NSString * const CYRequestURL;
```
   + 3> 在pch中包含CYConst.h文件

```objc
#import "CYConst.h"
```
---------------------------------

- **static :**
  + 1> 被static修饰的全局变量\常量
      + 1) 仅限于当前文件访问
      + 2) 改变了作用域
  + 2> 被static修饰的局部变量
      + 1) 只会占用一块内存，在整个程序运行过程都不会销毁，只会初始化一次
      + 2) 改变了生命周期，并没有改变作用域
  + 例如：static int const money = 30;
    + 当前文件中你都可以访问，其它文件不行
- **const :**
- const只修饰它右边的内容，被const修饰的内容都是常量、都是不能再修改的

```objc
            int * const p1;  p1是常量，*p1是变量
            int const * p2;  p2是变量，*p2是常量
            const int * p3;  p3是变量，*p3是常量
            const int * const p4;  p4是常量，*p4是常量
            int const * const p5;  p5是常量，*p5是常量

```
- extern : 可以引用一个全局变量\常量
- 今后能用const就用const
- 只有一块内存，不能被修改
- **什么时候使用宏什么时候全局变量呢**
  + 在程序运行过程中，未确定的那些值，但是又可能经常被用到，就用宏（例如前面定义了一个全局颜色）
  + 而已经完全确定，固定的，全世界又要用的，就用const的全局变量
