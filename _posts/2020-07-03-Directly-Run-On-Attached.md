---
title: VS 中一键 Attach 到指定进程
author: Yohiro
date: 2020-07-03
categories: [Debug]
tags: [unrealengine, visualstudio, extension, tool]
render_with_liquid: false
img_path: /assets/images/DirectlyRunOnAttached/
---
## 背景

调试的过程中，有时会碰到要 Attach 到某个进程的情况。遇到这种情况，一般来说就是 Alt+D,P 呼出 AttachToProcess 界面，然后找到进程 Attach。今天发现一个方法，可以一键直接开启 Attach 到指定的进程，而且具有一定的扩展性。本着好东西大家一起分享的原则，现记录如下。

## 原理

这个方法是利用 VS 的宏和库，将指令脚本化，类似 Excel 里的宏。通过引入VS的扩展 **Visual Commander** 执行相应的命令。

## 步骤

1. `VS 菜单栏-> Extensions -> Manage Extensions` 搜 Visual Commander（下称 VCmd），先把它下下来<br>
  ![Vcmd_Extension](VisualCommander.png){: .light .shadow .rounded-10}

2. 下载完成后，可在 Extensions 中找到 VCmd，找到 Commands，我们可以在这里添加自定义的指令<br>
  ![VCmd_Command_Intro](VCMDWhere.png){: .light .shadow .rounded-10}

3. 然后点击 Add 以添加编辑页面，语言可选 C# 或 VB,可以看到用了 DTE 接口。可以起个名字，方便管理<br>
  ![VCmd_Command_View](Command.png){: .light .shadow .rounded-10}

4. 要运行的内容，按照我们的需求，就是找到指定的进程并 Attach 上去。我这里 Attach 的是 AutomationTool

    ```csharp
    using EnvDTE;
    using EnvDTE80;

    public class C : VisualCommanderExt.ICommand
    {
        public void Run(EnvDTE80.DTE2 DTE, Microsoft.VisualStudio.Shell.Package package)
        {
            foreach(Process2 proc in DTE.Debugger.LocalProcesses)
            {
                if(proc.Name.ToString().Contains("AutomationToolLauncher.exe"))
                {
                    proc.Attach2("");
                    return;
                }
            }
            System.Windows.MessageBox.Show("AutomationToolLauncher not found.");
        }
    }
    ```

    记得保存。

5. 这个时候已经可以运行了，但是为了一键 Attach，我们得把它绑定到一个按键上
  菜单栏找 `Tools -> Customize -> Commands` 标签下，选择 `ToolBar -> Standard`，这里看个人习惯，我放到了工具栏上 Standard 栏，右侧 `Add Command` 找到要执行的 Command<br>
    ![Add_Customize_Command](AddCommand.png)<br>
  Extensions 目录下找到相应的 Command，不记得是第几个可以在 VCmd 的 Command 编辑界面看看左上角

6. 然后 Preview 里面选中后可以移动到合适的位置或者 `Modify Selection` 改个喜欢的名字<br>
    ![Modify_Customize_Command](Customize.png){: .light .shadow .rounded-10}
  
7. 当目标进程启动时，直接点我们添加的按钮，就可以 Attach 到了<br>
    ![Customize_Button](CustomizeAttachButton.png){: .light .shadow .rounded-10}

## 最后

其他的一些功能和样例可以参考 Extensions -> VCmd -> Command Examples/ Extension Examples 或者前往 [**visual-commander**](https://vlasovstudio.com/visual-commander/commands.html)