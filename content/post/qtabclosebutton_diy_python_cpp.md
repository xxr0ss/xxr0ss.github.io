---
title: "QTabWidget/QTabBar的自定义关闭按钮实现 Python & C++"
date: 2021-02-19T10:53:46+08:00
ShowToc: true
---

QTabWidget和QTabBar两个类的设计上，都提供了`setTabIcon()`用于设定指定一个页面（Tab）的图标。这个图标是很容易修改的。但是Tab如果是可关闭的，那么会显示一个关闭按钮。这个按钮的图标可以通过qss，也就是`tabbar.setStyleSheet()`实现：
`QTabBar::close-button{ image: url(url_to_image) }`
当然，还有`QTabBar::close-button::hover`这样的可以用于进一步细化。更多内容就得去看style sheet的文档了。

但如果，出于一些原因，我们非要通过qrc来实现呢，或者想要自定义这个按钮的绘制过程，要如何着手？答案是研究源码。
`QTabWidget`和`QTabBar`的文档里，我并没有看到提供了方便的函数用于实现我的这一想法。

我本来是在使用PySide6时发现这个问题的，但是解决过程中，同时看了Qt的C++实现部分和Python实现部分的源码，所以理论上无论你是用Python还是C++在进行Qt开发，如果你碰到和我一样的问题，相信这篇文章能帮助到你。

先说明两个类的关系，两个类都继承于`QWidget`，`QTabWidget`内部保存了一个`QTabBar`, 本身可以理解为组合了`QTabBar`和`QStackedWidget`，这一点源码中可以得到印证：
```cpp
class QTabWidgetPrivate : public QWidgetPrivate
{
    Q_DECLARE_PUBLIC(QTabWidget)
public:
	// ...
    QTabBar *tabs;
    QStackedWidget *stack;
	// ...
};
```
由于上述关系，另外`QTabWidget`也提供public方法用于获取内部保存的`QTabBar`，所以后面就不提`QTabWidget`了，**我们把关注点放在`QTabBar`上**。


QTabBar可以设置指定`index`的是否启用关闭按钮，可以使用`setTabButton()`来设置按钮。所以我们源码从这些线索顺着找，我机子上源码位置是`D:\Qt\6.0.1\Src\qtbase\src\widgets\widgets\qtabbar.cpp`
源码里`QTabBar::insertTab()`，`QTabBar::setTabsClosable()`内部使用了`setTabButton()`。可以看到源码里还一个`CloseButton`类继承自`QAbstractButton`类。（这里不放源码了，篇幅有限）

分析两处调用过程，**共同点**在于：
* `setTabButton()`使用了创建出来的`CloseButton`对象
* `CloseButton`的`ButtonPosition`都来自QTabBar的
	`(ButtonPosition)style()->styleHint(QStyle::SH_TabBar_CloseButtonPosition, nullptr, this)`
* 连接了`CloseButton`的`clicked()`信号和QTabBar的`_q_closeTab()`槽
第2点原因是，QTabBar可以设置关闭按钮位置，所以这里需要获取其位置。

既然有源码，我们照着改就行，我是PySide6进行开发，所以是进行C++到Python的移植（我没有用别的命名，用的是和`QTabBar`里面一样的命名）：
```python
class CloseButton(QAbstractButton):
    def __init__(self, parent: QWidget):
        super(CloseButton, self).__init__(parent)
        self.parent = parent
        self.setFocusPolicy(Qt.NoFocus)
        self.resize(self.sizeHint())
        self.setEnabled(True)
        self.clicked.connect(self.log_clicked)

    @Slot()
    def log_clicked(self):
        print('Close Button clicked')

    def sizeHint(self) -> QSize:
        self.ensurePolished()
        width = self.style().pixelMetric(QStyle.PM_TabCloseIndicatorWidth, None, self)
        height = self.style().pixelMetric(QStyle.PM_TabCloseIndicatorHeight, None, self)
        return QSize(width, height)

    def minimumSizeHint(self) -> QSize:
        return self.sizeHint()

    def enterEvent(self, event: QEnterEvent) -> None:
        if self.isEnabled():
            self.update()
        QAbstractButton.enterEvent(self, event)

    def leaveEvent(self, event: QEvent) -> None:
        if self.isEnabled():
            self.update()
        QAbstractButton.leaveEvent(self, event)

    def paintEvent(self, event: QPaintEvent) -> None:
        option = QStyleOption()
        option.initFrom(self)
        if self.isEnabled() and self.underMouse() and not self.isCheckable() and not self.isDown():
            option.state |= QStyle.State_Raised
        if self.isChecked():
            option.state |= QStyle.State_On
        if self.isDown():
            option.state |= QStyle.State_Sunken

        tb: QTabBar = self.parent
        if isinstance(tb, QTabBar):
            index = tb.currentIndex()
            position = TabsManager.side_enum[tb.style().styleHint(QStyle.SH_TabBar_CloseButtonPosition, None, tb)]
            if tb.tabButton(index, position):
                option.state |= QStyle.State_Selected

        p = QPainter(self)
        self.style_draw(option, p)

    def style_draw(self, option: QStyleOption, p: QPainter):
        # 移植PE_IndicatorTabClose的绘制逻辑, qcommonstyle.cpp里有个大switch-case对C++ QTabBar源码里面，CloseButton的paintEvent最后面的style()->drawPrimitive第一个参数就是处理这个，下面的代码相当于是移植了switch-case里PE_IndicatorTabClose的代码
        size = self.style().proxy().pixelMetric(QStyle.PM_SmallIconSize, option)
        mode: QIcon.Mode = (QIcon.Active if option.state & QStyle.State_Raised else QIcon.Normal) \
            if option.state & QStyle.State_Enabled else QIcon.Disabled
        if not option.state & QStyle.State_Raised \
                and not option.state & QStyle.State_Sunken \
                and not option.state & QStyle.State_Selected:
            mode = QIcon.Disabled

        state: QIcon.State = QIcon.On if option.state & QStyle.State_Sunken else QIcon.Off
        pixmap = CloseButton.tabBar_close_button_icon().pixmap(QSize(size, size), self.devicePixelRatio(), mode, state)
        self.style().proxy().drawItemPixmap(p, option.rect, Qt.AlignCenter, pixmap)

    @staticmethod
    def tabBar_close_button_icon() -> QIcon:
        # qcommonstyle.cpp里获取Pixel的逻辑改了，现在这里的代码实现是用了我自己.qrc里的资源。
        icon = QIcon()
        # add
        icon.addPixmap(QPixmap(':/default/icons/ui/closeButton.png'), QIcon.Normal, QIcon.Off)
        icon.addPixmap(QPixmap(':/default/icons/ui/closeButton_down.png'), QIcon.Normal, QIcon.On)
        icon.addPixmap(QPixmap(':/default/icons/ui/closeButton_hover.png'), QIcon.Active, QIcon.Off)
        return icon
```
相当一部分是找到对应源码理解后去移植，参考到的源码大致有
`D:\Qt\6.0.1\Src\qtbase\src\widgets\widgets\qtabbar.cpp`
`D:\Qt\6.0.1\Src\qtbase\src\widgets\styles\qcommonstyle.cpp`

里面的`TabsManager.side_enum`是我定义的别的类用于对QTabWidget进行管理，内容是：
```python

side_enum = [QTabBar.ButtonPosition.LeftSide, QTabBar.ButtonPosition.RightSide]
```
（不像C++里面枚举可以当成int，所以为了能调用必要的api，就手动把`int`转成对应的`ButtonPosition`）。
还有就是实现了自定义的`close_button_icon`，把几种状态的图标都加进去了（正常/hover/ down)。

实现了自定义的`CloseButton`后，在添加Tab的时候，我编写的`TabsManager`里处理逻辑是：
```python
    def add_editor_tab(self, widget: QWidget, title: str):
        # automatically remove welcome page by default when opened new tab
        if self._tabs.count() == 1:
            if isinstance(self._tabs.widget(0), WelcomePage):
                self._tabs.removeTab(0)
        if isinstance(widget, CodeEditorWidget):
            # mark tab as modified when content of editor changed
            widget.content_status_changed.connect(
                lambda need_saving: self.update_tab_status(widget, need_saving))

        idx = self._tabs.addTab(widget, title)
        self._tabs.setCurrentIndex(idx)

        # use customized Close Button
        close_side = self.side_enum[widget.style().styleHint(
            QStyle.SH_TabBar_CloseButtonPosition, None, widget)]
        self._tabs.tabBar().setTabButton(idx, close_side, btn := CloseButton(self._tabs.tabBar()))
        btn.clicked.connect(self._q_closeTab)
        
    def _q_closeTab(self):
    	# 模仿源码里进行的实现
        btn: CloseButton = self.sender()
        tab_bar: QTabBar = self.tabs.tabBar()
        tab_to_close = -1
        close_side = self.side_enum[tab_bar.style().styleHint(QStyle.SH_TabBar_CloseButtonPosition, None, tab_bar)]
        for i in range(tab_bar.count()):
            if tab_bar.tabButton(i, close_side) == btn:
                tab_to_close = i
                break
        if tab_to_close != -1:
            self._tabs.tabCloseRequested.emit(tab_to_close)
```
这里值得注意的点在最后我设置自定义`CloseButton`后，手动连接了自定义的`CloseButton`的`clicked`信号和`QTabWidget`的`tabCloseRequest`信号，必须要加上这一行。前面代码有提到那几步里面连接`QTabBar`的`_q_closeTab()`槽，我自己实现了一遍。

总结一下自定义关闭按钮的过程：
* 照着源码根据自己需求编写自己的`CloseButton`
	* 注意处理绘制逻辑
	* 注意处理图标状态
* 在`QTabBar`/`QTabWidget`添加tab的时候，在这个位置（`index`）使用自定义的`CloseButton`
	* 注意要连接`clicked`信号和`tabCloseRequested`信号


最后，分享一下这些代码涉及到的[源码](https://github.com/xxr0ss/pyQEditor)
