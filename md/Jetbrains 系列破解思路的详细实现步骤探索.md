> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn//thread-1921814-1-1.html)

> [md] 参考最近论坛很火的一个帖子：[JetBrains 全家桶系列 2024 破解思路](https://www.52pojie.cn/thread-1919098-1-1.html) 该帖子作者：[V......

 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) 细雨微风 _ 本帖最后由 细雨微风 于 2024-5-8 17:27 编辑_  

参考最近论坛很火的一个帖子：  
[JetBrains 全家桶系列 2024 破解思路](https://www.52pojie.cn/thread-1919098-1-1.html)  
该帖子作者：[Vvvvvoid](https://www.52pojie.cn/home.php?mod=space&uid=375476)

该帖子没有直接提供破解文件，看了该帖子，我自己想动手实现一番。下面是我总结的过程，其中用到了不少 “知识”，可供参考。

看帖子的说明和代码，可知：破解的原理是找到一个 IDEA 在启动后就会加载的 jar 包和里面相关的类，然后反编译修改这个类（添加 static 代码），这样 IDEA 启动后就会执行我们添加的静态代码块。在静态代码块中，找到所有的窗口（Window），然后遍历得到所有的按钮，如果按钮文字包含 “Quit” 或“Close”字样，表示它是激活界面的按钮，然后我们清除这个按钮的原本事件监听，再添加自定义的事件监听，实现让点击 “Quit” 按钮后，关闭该激活对话框，达到正常使用 IDEA 功能的效果。

整个原理非常简单，很多评论感叹说 “这样竟然都可以实现破解”，这确实是一个不错的思路，怎么前人没有想到呢！

下面是我的探索步骤。

### Step1: 下载最新 IDEA 研究

为了研究，我在虚拟机 Linux Mint 中，用 IntelliJ 的 Toolbox 工具下载了最新的 Linux 版的 IDEA Ultimate 2024.1 版本。发现该软件现在又可以免登录账号试用 30 天了。不过我现在为了破解，恰恰不想要这个试用，因此我把 Linux 系统的时间调整到了 2025 年 5 月 1 日，也就是一年后，再打开 IDEA ，果然提示试用已过期，让我激活。当然由于时间错误，IDEA 也会提醒 https 的证书问题，点击确认即可。

### Step2: 寻找合适的 jar 包和类

要修改 IDEA 中的代码，从哪里入手呢？看帖子可知，要找一个 IDEA 启动时依赖的 jar 包，并且要找到这个 jar 包中一个被加载的类进行修改。

于是我在 IDEA 安装目录的 lib 目录中，找了一个 util.jar ，因为看名字它应该会被加载，而且这个包不大，比较合适。再通过 7zip 打开浏览包，只能大致看看类路径和名字，并不能看到各个类的复杂程度，还要借助反编译工具。

帖子使用了 [jadx](https://github.com/skylot/jadx)，因此我也下载了 jadx 使用。通过 jadx 打开 util.jar，最终看中了 com.intellij.util.UrlImpl.class 这个类（接口不行，无法添加 static 静态代码块），直觉上这个类是会被加载使用的，而且这个类的反编译代码比较少，比较简单，也没有使用 Kotlin(目前我对 Kotlin 不熟悉)，更没有混淆代码，因此选择了这个类进行修改研究。

于是我把 util.jar 从虚拟机中复制到 Windows 宿主机中准备研究。

例如，我把 util.jar 放在了 “D:\myCrack” 目录中，这个目录就是我专门进行现在研究的工作目录。

后续事实证明，选择这个类是可行的。

### Step3: 简单验证：反编译并修改代码运行 IDEA

要修改代码，首先要反编译 class 得到代码。在 jadx 中就能看到 UrlImpl.class 这个类反编译后得到的代码。

如果想把这个 class 文件从 jar 提取出来单独研究，可使用下面的命令（参考了帖子的脚本命令）：

```
jar -xvf util.jar com/intellij/util/UrlImpl.class

```

建议运行上面的命令，这样就在 myCrack 目录中创建了 com/intellij/util 目录，并提取出了 UrlImpl.class 文件。

现在，我们在 com/intellij/util 目录创建一个 UrlImpl.java 空文件，把反编译后的代码复制到 UrlImpl.java 文件中。

现在，就开始添加 static 代码了。先和帖子一样，进行简单验证，看代码是否能被执行，于是在 UrlImpl.java 中类定义的开头部分，添加静态代码块，进行一行输出，这样添加在开头是比较清晰的，添加后的代码类似这样：

```
import xxx; // import 部分目前不用修改

public final class UrlImpl implements Url {

    // 帖子中的 static 静态代码块
    static {
        System.out.println(">>>>>>>     WindowWatcher");
    }

    // 以下省略... 即反编译后原有的代码...
}

```

这样，java 代码修改好了，下一步就是重新编译生成该文件的 class 文件了。如果直接运行：

```
javac -encoding UTF-8 com/intellij/util/UrlImpl.java

```

(说明，java 文件统一使用 UTF-8 编码，请自行设置 java 编辑器编码。而且，建议使用的 JDK 版本和 IDEA 使用的版本是一致的，例如现在 IDEA 使用的 JDK 版本是 17)

则会报错，因为源文件中依赖到的类不存在，编译时会报错。于是，我看了下代码，发现一是缺少 util.jar 本身的类；二是缺少 @nullable 、 @NotNull 这些注解，浏览一下，发现这些注解就在 IDEA 安装目录下 lib 目录的 annotations.jar 包中。这样再分析，一些缺少的类在 lib 目录下查找，因此还需要 util_rt.jar 、 util-8.jar 这两个包。

那么，编译的时候就要把这些 jar 包加入到 classpath 中，可以使用 -cp 命令行参数，多个 jar 包使用分号（;）进行分割。因此，除了 util.jar 包外，我们再把 annotations.jar 、 util_rt.jar 和 util-8.jar 这三个文件复制到 myCrack 目录中，这样，就可以使用如下命令编译 UrlImpl.java 文件了。

```
javac -encoding UTF-8 -cp util.jar;annotations.jar;util_rt.jar;util-8.jar com/intellij/util/UrlImpl.java

```

经过此，会在 com/intellij/util 生成并覆盖原 UrlImpl.class 文件。  
然后，可以使用如下命令，把 class 文件再覆盖打包到 util.jar 中，执行如下命令：

```
jar -uvf util.jar com/intellij/util/UrlImpl.class

```

这样， util.jar 就被重新生成了。把这个 util.jar 替换到 Linux 中的 IDEA 的 lib 目录中，仍可以让 IDEA 正常运行。

但是，如果想看到我们的输出，则需要进入 IDEA 安装目录的 bin 目录，通过控制台启动 IDEA ：

```
./idea.sh 

```

这样，启动后一段时间，即可看到控制台输出 “>>>>>>>     WindowWatcher”，说明我们替换文件再运行成功了！也证明选择 UrlImpl.class 文件应该是可行的。

### Step4: 按帖子思路实现破解

实现了上面步骤，现在就是要根据帖子的思路，编写静态代码块，用线程实现：获取当前窗口，再获取 “Quit IntelliJ IDEA” 按钮，改变其监听事件的代码。  
帖子中的代码还是有点问题的，不能直接复制添加到 UrlImpl.java 中编译，它第一缺少 import 相关 awt 和 swing 的类（虽然说这个好解决）；第二缺少 watcherFlag 变量的定义（这个也好解决，定义一个 boolean 类型变量即可）；第三缺少 getButtons() 方法。可以看出，getButtons() 方法就是获取 JDialog 中所有的按钮的，实现这个方法看似难度也不大。从这也可看出，这个帖子确实只是提供了思路，还需要自己有一定的动手和思考能力，特别是利用搜索引擎甚至是现在 ChatGPT 的能力（后续可以看出）。

但是，如果让我在 Notepad++ 中编写修改上面说的缺少的这三点，还是麻烦的，最好是可以在 IDEA 中进行编写，还能提示我。怎么办呢？我想到的方法就是新建一个项目，然后把 util.jar;annotations.jar;util_rt.jar;util-8.jar 这四个包加入到项目的依赖，这样，在项目中新建一个 CrackIdea.java ，在里面修改代码，则会有提示。

一开始，我解决了前两个缺少的问题，第三个问题按照 “获取 JDialog 中所有的 JButton 按钮” 通过搜索大概写了一下。但发现并不能起到作用。这时正好发现帖子有人回复做出了 “成品”，他是和帖子一样修改 com/jetbrains/ls/responses/License.class 类的，因此他提供修改的是 product-client.jar 文件。但我把这个 jar 文件放到 Linux 虚拟机的 IDEA 最新版本中，并不能正常运行，应该是版本问题。

于是我又研究他的 product-client.jar 文件，反编译查看代码，看到了他的完整的实现代码（包含 getButtons() 方法）：

```
package com.jetbrains.ls.responses;

import java.awt.Window;
import java.awt.event.ActionEvent;
import java.awt.event.ActionListener;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import javax.swing.JButton;
import javax.swing.JDialog;
import org.jetbrains.annotations.NotNull;
import org.jetbrains.annotations.Nullable;

public class License extends LicenseOld {

    static boolean watcherFlag = true;

    // 省略了部分原有代码

    static {
        System.out.println(">>>>>>>     WindowWatcher");
        Thread watcherThread = new Thread(() -> {
            while (watcherFlag) {
                try {
                    Thread.sleep(5000L);
                    for (JDialog jDialog : Window.getWindows()) {
                        if (jDialog instanceof JDialog) {
                            final JDialog dialog = jDialog;
                            List<JButton> buttons = getButtons(dialog);
                            for (JButton button : buttons) {
                                String text = button.getText();
                                System.out.println("Button Text: " + text);
                                dialog.setTitle("K'ed by: marlkiller");
                                if (text != null && (text.contains("Quit") || text.contains("Close"))) {
                                    dialog.setTitle(dialog.getTitle() + " 请点击[" + text + "] 按钮");
                                    button.removeActionListener(button.getActionListeners()[0]);
                                    button.setText("<Close>");
                                    button.addActionListener(new ActionListener() { // from class: com.jetbrains.ls.responses.License.1
                                        public void actionPerformed(ActionEvent e) {
                                            License.watcherFlag = false;
                                            dialog.setVisible(false);
                                        }
                                    });
                                }
                            }
                        }
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        watcherThread.start();
    }

    private static List<JButton> getButtons(JDialog dialog) {
        List<JButton> bs = new ArrayList<>();
        for (JButton jButton : dialog.getComponents()) {
            if (jButton instanceof JButton) {
                bs.add(jButton);
            }
        }
        return bs;
    }
}

```

于是，我把他的 getButtons() 代码移到我的 UrlImpl.java 中，重新编译 java 替换 class 到 jar 中，放入最新版 IDEA 中通过控制台运行查看输出信息和实际界面，仍无效果。

然后我一次次通过添加 System.out.println() 语句，借输出信息进行 “调试”（要是真有手段能真正的调试就好了，可惜没找到方法），发现问题还是出在 getButtons() 中，因为 System.out.println("Button Text:" + text); 这行代码我在运行时并没有输出 “Quit IntelliJ IDEA” 这些按钮。虽然该大神写的方法在我这里没法成功，但至少提供了思路，就是 IDEA 确实是基于 Swing 实现的，而且 getButtons() 方法的作用就是获取 dialog 中所有的 JButton 按钮。所以，我又尝试输出每个 window 中的所有 Component 组件，一层层的找下去，不过，弄了好久都没正确实现，原因可能是它这个 dialog 嵌套的组件太多了，而且根据输出信息来寻找组件也比较麻烦。

怎么办？我突然想到可以使用 ChatGPT 来写代码，于是我提示 ChatGPT ： “Java 如何获取 Component 里面的按钮对象”，此时 ChatGPT 返回：

> 要获取一个 Component 中的所有按钮对象，可以使用递归遍历的方法，检查 Component 中的每个子组件，如果子组件是按钮对象，则将其添加到列表中。以下是一个示例代码：
> 
> ```
> import java.awt.Component;
> import java.awt.Container;
> import java.awt.Window;
> import java.awt.Button;
> import java.util.ArrayList;
> import java.util.List;
> 
> public class ComponentUtils {
> 
>     public static List<Button> getButtons(Component component) {
>         List<Button> buttons = new ArrayList<>();
>         if (component instanceof Button) {
>             buttons.add((Button) component);
>         } else if (component instanceof Container) {
>             Component[] children = ((Container) component).getComponents();
>             for (Component child : children) {
>                 buttons.addAll(getButtons(child));
>             }
>         }
>         return buttons;
>     }
> 
>     public static void main(String[] args) {
>         // 示例用法：获取窗口中的所有按钮
>         Window window = new Window(null); // 创建一个空窗口
>         Button button1 = new Button("Button 1");
>         Button button2 = new Button("Button 2");
>         window.add(button1);
>         window.add(button2);
> 
>         List<Button> buttons = getButtons(window);
>         for (Button button : buttons) {
>             System.out.println(button.getLabel());
>         }
>     }
> }
> 
> 
> ```

可以看出，ChatGPT 使用了递归！层层嵌套使用递归，这是多么简单的道理！于是我修改一下，应用到我的 UrlImpl.java 中。此时，我通过观察、调试和分析，也发现帖子中的代码有几处优化之处：1. 可以直接通过 dialog 标题为 “Licenses” 来判定选择处理激活对话框；2. 可直接通过按钮文字 “Quit IntelliJ IDEA” 来判定选择处理推出按钮；3. 可循环 ActionListeners 来清除原有按钮监听事件；4. 在新添加的按钮事件中，把对话框设置为 dispose() 即销毁对话框，而不是 hidden() 隐藏，这样可能更节省资源；4. 把 watcherFlag = false; 放到监听器外面，这样只要我们找到了按钮并处理了监听事件，这个线程就可以结束了，而放到里面的话，必须等到点击完按钮线程才会结束。同时，在各个 for 循环中合适的使用 break; 避免无效循环。

综上，我优化后的 UrlImpl.java 类如下：

```
package com.intellij.util;

import com.intellij.openapi.ui.Messages;

import javax.swing.*;
import java.awt.*;
import java.awt.event.ActionListener;
import java.util.ArrayList;
import java.util.List;
// 省略了类原有的 import 部分，自己添加

public final class UrlImpl implements Url {

    private static boolean watcherFlag = true;

    static {
        System.out.println("**********    util.jar; Addition of WindowWatcher");

        // 新建线程运行
        new Thread(() -> {
            while (watcherFlag) {
                try {
                    System.out.println("进入 WindowWatcher 循环...");
                    Thread.sleep(5000L);
                    // 遍历 Windows
                    for (Window window : Window.getWindows()) {
                        if (window instanceof JDialog) {
                            final JDialog dialog = (JDialog) window;
                            if ("Licenses".equals(dialog.getTitle())) {
                                List<JButton> buttons = getButtons(dialog);
                                for (JButton button : buttons) {
                                    String buttonText = button.getText();
                                    System.out.println("Button Text: " + buttonText);

                                    if ("Quit IntelliJ IDEA".equals(buttonText)) {
                                        // 取消 button 原有的 ActionListeners
                                        for (ActionListener actionListener : button.getActionListeners()) {
                                            button.removeActionListener(actionListener);
                                        }

                                        // 设置允许重新试用
                                        button.addActionListener(e -> {
                                            // 此方法用于提醒，因为后续再点击 Register 的话，由于此线程结束了，不会再处理新打开的激活窗口的监听事件了
                                            Messages.showInfoMessage("后续请勿打开 Help - Register 菜单", "提醒");
                                            dialog.dispose(); // 即主动关闭该对话框
                                        });

                                        button.setText("<Close>"); // 重新设置 text
                                        dialog.setTitle("若要继续试用，请点击[" + button.getText() + "]按钮");

                                        // 设置标记变量为 false，表示无需再循环设置此试用代码了
                                        watcherFlag = false;
                                        // 后续不用执行
                                        break;

                                    }
                                }
                                // 此处也 break;
                                break;
                            }
                        }
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }

            System.out.println("**********    util.jar; WindowWatcher Execution Done!");
        }).start();

    }

    // 此方法获取 dialog 中所有 JButton
    private static List<JButton> getButtons(JDialog dialog) throws InterruptedException {
        List<JButton> bs = new ArrayList<>();
        for (Component component : dialog.getComponents()) {
            bs.addAll(getNestButtons(component));
        }
        return bs;
    }

    // 此方法是基于 ChatGPT 提供的方法修改而来
    public static List<JButton> getNestButtons(Component component) {
        List<JButton> buttons = new ArrayList<>();
        if (component instanceof JButton) {
            buttons.add((JButton) component);
        } else if (component instanceof Container) {
            Component[] children = ((Container) component).getComponents();
            for (Component child : children) {
                buttons.addAll(getNestButtons(child));
            }
        }
        return buttons;
    }

    // 省略了原有类代码

}

```

由于其中使用了 IDEA 的 Messages 进行提醒，因此还要引入 app-client.jar 类。

然后，使用命令（添加了 app-client.jar）：

```
javac -encoding UTF-8 -cp util.jar;annotations.jar;util_rt.jar;util-8.jar;app-client.jar com/intellij/util/UrlImpl.java

```

编译类，发现生成了两个类：UrlImpl.class 和 UrlImpl$1.class，这是由于我们使用了匿名内部类处理事件的结果。所以，要把这两个类都加入到 util.jar 中进行覆盖。通过咨询 ChatGPT，可使用如下命令加入多个 class 文件到 jar 中：

```
jar -uvf util.jar com/intellij/util/UrlImpl.class com/intellij/util/UrlImpl$1.class

```

即使用空格分隔即可。  
这样，我们就得到了新的 util.jar 文件，此时，放入 IDEA 中，发现果然起效果了！

所以，还是需要自己有一定的动手能力。

### Step5：优化思路

我又思考了一下，何必要大费周章找到按钮来修改事件呢，直接修改激活对话框 JDialog 的窗口关闭事件不就可以了吗？

于是，破解代码就更简单了，主要代码如下：

```
public class UrlImpl {
    private static boolean watcherFlag = true;

    static {
        System.out.println("**********    util.jar; Addition of WindowWatcher");

        // 新建线程运行
        new Thread(() -> {
            while (watcherFlag) {
                try {
                    System.out.println("进入 WindowWatcher 循环...");
                    Thread.sleep(500L);
                    // 遍历 Windows
                    for (Window window : Window.getWindows()) {
                        if (window instanceof JDialog) {
                            final JDialog dialog = (JDialog) window;
                            if ("Licenses".equals(dialog.getTitle())) {

                                for (WindowListener windowListener : dialog.getWindowListeners()) {
                                    dialog.removeWindowListener(windowListener);
                                }

                                dialog.addWindowListener(new WindowAdapter() {
                                    @Override
                                    public void windowClosing(WindowEvent e) {
                                        Messages.showInfoMessage("后续请勿打开 Help - Register 菜单", "提醒");
                                        dialog.dispose();
                                    }

                                });

                                dialog.setTitle("若要继续试用，请点击*右上角*关闭按钮");

                                // 设置标记变量为 false，表示无需再循环设置此试用代码了
                                watcherFlag = false;
                                // 后续不用执行
                                break;
                            }

                        }
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }

            System.out.println("**********    util.jar; WindowWatcher Execution Done!");
        }).start();

    }
}

```

无需写比较繁琐的寻找按钮的方法！读者应该能很轻松的自己动手实现。

### 后续

之后，我又研究了 javaagent 方法，实现方式会更加简单，过几天再分享吧，写了这么多太累啦。![](https://avatar.52pojie.cn/data/avatar/000/37/54/76_avatar_middle.jpg)Vvvvvoid 1. 最好找一个没有第三方依赖的, 那些注解相关的删了也无所谓的;  
2. 获取 button 确实用的递归, 当时代码帖漏了 (gpt 帮我写的)  
3.boolean watcherFlag 这个变量需加上 volatile 修饰, 增加可见性, 防止从高速缓冲读取变量 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) jacktvt 既然可以通过修改系统时间来让试用过期，那么是不是也可以每次启动 Idea 返回一个特定时间达到无限试用的效果 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) havonz jetbrains 似乎没有把精力花在反破解这件事情上 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) lypxynok 谢谢楼主，学习了 ![](https://avatar.52pojie.cn/data/avatar/000/88/33/34_avatar_middle.jpg) px307 向大佬学习，谢谢分享![](https://static.52pojie.cn/static/image/smiley/default/40.gif) ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) beichen1031 有无版本限制？![](https://avatar.52pojie.cn/images/noavatar_middle.gif)moresun 新的思路，学习了大佬寻找、编译类的过程，很强 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) wubo777 Navicat Premium 16 破解思路也研究研究![](https://static.52pojie.cn/static/image/smiley/laohu/laohu34.gif) ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) wasm2023

> [wubo777 发表于 2024-5-9 10:25](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=50314125&ptid=1921814)  
> Navicat Premium 16 破解思路也研究研究

这个可以有 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) linkam 学到了，有空研究研究