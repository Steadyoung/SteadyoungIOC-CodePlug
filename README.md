# SteadyoungIOC-CodePlug注解框架快速生成代码插件

## ![LOGO图片](https://upload.jianshu.io/collections/images/1633997/%E4%B8%8B%E8%BD%BD.png?imageMogr2/auto-orient/strip|imageView2/1/w/240/h/240)  [SteadyoungIOC注解框架专栏](https://www.jianshu.com/c/3734b4eb3d17)

博客解说：[Android Studio插件开发-SteadyoungIOC注解生成器：Steadyoung-CodePlug](https://www.jianshu.com/p/dea6b3388709)

注解框架：[自己简易打造的IOC注解框架：SteadyoungIOC](https://www.jianshu.com/p/0c11f3f27ddc)

注解框架源码：[SteadyoungIOC-CodePlug](https://github.com/Steadyoung/SteadyoungIOC-CodePlug)

插件下载：[SteadyoungIOC-CodePlug.jar](https://raw.githubusercontent.com/Steadyoung/SteadyoungIOC-CodePlug/master/SteadyoungIOC-CodePlug.jar)

插件可以使用 Alt + Insert 智能插入 呼出，也可以使用 Ctrl + Shift + I 或者 Ctrl + Shift + Alt + I 快捷键： 

![SteadyoungIOC框架代码生成演示](https://upload-images.jianshu.io/upload_images/8541415-8ae31adc7d03241f.png)

- 获取光标所在行的布局文件 --> R.layout.xxxx.xml；

```
    /**
     * 获取当前光标的layout文件
     */
    private String getCurrentLayout(Editor editor) {
        Document document = editor.getDocument();
        CaretModel caretModel = editor.getCaretModel();
        int caretOffset = caretModel.getOffset();
        int lineNum = document.getLineNumber(caretOffset);
        int lineStartOffset = document.getLineStartOffset(lineNum);
        int lineEndOffset = document.getLineEndOffset(lineNum);
        String lineContent = document.getText(new TextRange(lineStartOffset, lineEndOffset));
        String layoutMatching = "R.layout.";
        if (!TextUtils.isEmpty(lineContent) && lineContent.contains(layoutMatching)) {
            // 获取layout文件的字符串
            int startPosition = lineContent.indexOf(layoutMatching) + layoutMatching.length();
            int endPosition = lineContent.indexOf(")", startPosition);
            String layoutStr = lineContent.substring(startPosition, endPosition);
            // 可能是另外一种情况 View.inflate
            if (layoutStr.contains(",")) {
                endPosition = lineContent.indexOf(",", startPosition);
                layoutStr = lineContent.substring(startPosition, endPosition);
            }
            return layoutStr;
        }
        return null;
    }
```

- 搜索整个项目获取到R.layout.xxxx.xml文件:

```
    @Override
    public void actionPerformed(AnActionEvent e) {
        // 获取project
        Project project = e.getProject();
        // 获取选中内容
        final Editor mEditor = e.getData(PlatformDataKeys.EDITOR);
        if (null == mEditor) {
            return;
        }
        SelectionModel model = mEditor.getSelectionModel();
        mSelectedText = model.getSelectedText();
        // 未选中布局内容，显示dialog
        if (TextUtils.isEmpty(mSelectedText)) {
            // 获取光标所在位置的布局
            mSelectedText = getCurrentLayout(mEditor);
            if (TextUtils.isEmpty(mSelectedText)) {
                mSelectedText = Messages.showInputDialog(project, "布局内容：（不需要输入R.layout.）", "未选中布局内容，请输入layout文件名", Messages.getInformationIcon());
                if (TextUtils.isEmpty(mSelectedText)) {
                    Util.showPopupBalloon(mEditor, "未输入layout文件名", 5);
                    return;
                }
            }
        }
        // 获取布局文件，通过FilenameIndex.getFilesByName获取
        // GlobalSearchScope.allScope(project)搜索整个项目
        PsiFile[] psiFiles = FilenameIndex.getFilesByName(project, mSelectedText + ".xml", GlobalSearchScope.allScope(project));
        if (psiFiles.length <= 0) {
            Util.showPopupBalloon(mEditor, "未找到选中的布局文件" + mSelectedText, 5);
            return;
        }
        XmlFile xmlFile = (XmlFile) psiFiles[0];
        List<Element> elements = new ArrayList<>();
        Util.getIDsFromLayout(xmlFile, elements);
        // 将代码写入文件，不允许在主线程中进行实时的文件写入
        if (elements.size() != 0) {
            PsiFile psiFile = PsiUtilBase.getPsiFileInEditor(mEditor, project);
            PsiClass psiClass = Util.getTargetClass(mEditor, psiFile);
            // 有的话就创建变量和findViewById
            if (mDialog != null && mDialog.isShowing()) {
                mDialog.cancelDialog();
            }
            mDialog = new FindViewByIdDialog(mEditor, project, psiFile, psiClass, elements, mSelectedText);
            mDialog.showDialog();
        } else {
            Util.showPopupBalloon(mEditor, "未找到任何Id", 5);
        }
    }
```

- 通过该布局文件去遍历找出含有id的布局标签，当然如果考虑完善一点需要考虑include等等:

```
    /**
     * 获取所有id
     *
     * @param file
     * @param elements
     * @return
     */
    public static java.util.List<Element> getIDsFromLayout(final PsiFile file, final java.util.List<Element> elements) {
        // To iterate over the elements in a file
        // 遍历一个文件的所有元素
        file.accept(new XmlRecursiveElementVisitor() {
            @Override
            public void visitElement(PsiElement element) {
            super.visitElement(element);
            // 解析Xml标签
            if (element instanceof XmlTag) {
                XmlTag tag = (XmlTag) element;
                // 获取Tag的名字（TextView）或者自定义
                String name = tag.getName();
                // 如果有include
                if (name.equalsIgnoreCase("include")) {
                    // 获取布局
                    XmlAttribute layout = tag.getAttribute("layout", null);
                    // 获取project
                    Project project = file.getProject();
                    // 布局文件
                    XmlFile include = null;
                    PsiFile[] psiFiles = FilenameIndex.getFilesByName(project, getLayoutName(layout.getValue()) + ".xml", GlobalSearchScope.allScope(project));
                    if (psiFiles.length > 0) {
                        include = (XmlFile) psiFiles[0];
                    }
                    if (include != null) {
                        // 递归
                        getIDsFromLayout(include, elements);
                        return;
                    }
                }
                // 获取id字段属性
                XmlAttribute id = tag.getAttribute("android:id", null);
                if (id == null) {
                    return;
                }
                // 获取id的值
                String idValue = id.getValue();
                if (idValue == null) {
                    return;
                }
                XmlAttribute aClass = tag.getAttribute("class", null);
                if (aClass != null) {
                    name = aClass.getValue();
                }
                // 添加到list
                try {
                    Element e = new Element(name, idValue,  tag);
                    elements.add(e);
                } catch (IllegalArgumentException e) {

                }
            }
            }
        });
        return elements;
    }
```

- 遍历完成后生成对话框，让用户可以自己选择需要生成注解的View以及点击事件，这个是Java GUI里面的内容，我直接百度找的代码实现了效果，贴出部分源码：

```
    /**
     * 解析mElements，并添加到JPanel
     */
    private void initContentPanel() {
        mContentJPanel.removeAll();
        // 设置内容
        for (int i = 0; i < mElements.size(); i++) {
            Element mElement = mElements.get(i);
            IdBean itemJPanel = new IdBean(new GridLayout(1, 4, 10, 10),
                    new EmptyBorder(5, 10, 5, 10),
                    new JCheckBox(mElement.getName()),
                    new JLabel(mElement.getId()),
                    new JCheckBox(),
                    new JTextField(mElement.getFieldName()),
                    mElement);
            // 监听
            itemJPanel.setEnableActionListener(this);
            itemJPanel.setClickActionListener(clickCheckBox -> mElement.setIsCreateClickMethod(clickCheckBox.isSelected()));
            itemJPanel.setFieldFocusListener(fieldJTextField -> mElement.setFieldName(fieldJTextField.getText()));
            mContentJPanel.add(itemJPanel);
            mContentConstraints.fill = GridBagConstraints.HORIZONTAL;
            mContentConstraints.gridwidth = 0;
            mContentConstraints.gridx = 0;
            mContentConstraints.gridy = i;
            mContentConstraints.weightx = 1;
            mContentLayout.setConstraints(itemJPanel, mContentConstraints);
        }
        mContentJPanel.setLayout(mContentLayout);
        jScrollPane = new JBScrollPane(mContentJPanel);
        jScrollPane.revalidate();
        // 添加到JFrame
        getContentPane().add(jScrollPane, 1);
    }
```

- 最后当用户点击确定生成最终的注解代码即可，主要生成注解@FindView(R.id.XXX)、@OnClick(R.id.XXX)、在OnCreate中生成SteadyoungIOC.jnject(this)等

```
    /**
     * 创建变量
     */
    private void generateFields() {
        for (Element element : mElements) {
            if (mClass.getText().contains("@FindView(" + element.getFullID() + ")")) {
                // 不创建新的变量
                continue;
            }
            // 设置变量名，获取text里面的内容
            String text = element.getXml().getAttributeValue("android:text");
            if (TextUtils.isEmpty(text)) {
                // 如果是text为空，则获取hint里面的内容
                text = element.getXml().getAttributeValue("android:hint");
            }
            // 如果是@string/app_name类似
            if (!TextUtils.isEmpty(text) && text.contains("@string/")) {
                text = text.replace("@string/", "");
                // 获取strings.xml
                PsiFile[] psiFiles = FilenameIndex.getFilesByName(mProject, "strings.xml", GlobalSearchScope.allScope(mProject));
                if (psiFiles.length > 0) {
                    for (PsiFile psiFile : psiFiles) {
                        // 获取src\main\res\values下面的strings.xml文件
                        String dirName = psiFile.getParent().toString();
                        if (dirName.contains("src\\main\\res\\values")) {
                            text = Util.getTextFromStringsXml(psiFile, text);
                        }
                    }
                }
            }

            StringBuilder fromText = new StringBuilder();
            if (!TextUtils.isEmpty(text)) {
                fromText.append("/****" + text + "****/\n");
            }
            fromText.append("@FindView(" + element.getFullID() + ")\n");
            fromText.append("private ");
            fromText.append(element.getName());
            fromText.append(" ");
            fromText.append(element.getFieldName());
            fromText.append(";");
            // 创建点击方法
            if (element.isCreateFiled()) {
                // 添加到class
                mClass.add(mFactory.createFieldFromText(fromText.toString(), mClass));
            }
        }
    }

    /**
     * 创建OnClick方法
     */
    private void generateOnClickMethod() {
        for (Element element : mElements) {
            // 可以使用并且可以点击
            if (element.isCreateClickMethod()) {
                // 需要创建OnClick方法
                String methodName = getClickMethodName(element) + "Click";
                PsiMethod[] onClickMethods = mClass.findMethodsByName(methodName, true);
                boolean clickMethodExist = onClickMethods.length > 0;
                if (!clickMethodExist) {
                    // 创建点击方法
                    createClickMethod(methodName, element);
                }
            }
        }
    }

    /**
     * 创建一个点击事件
     */
    private void createClickMethod(String methodName, Element element) {
        // 拼接方法的字符串
        StringBuilder methodBuilder = new StringBuilder();
        methodBuilder.append("@OnClick(" + element.getFullID() + ")\n");
        methodBuilder.append("private void " + methodName + "(" + element.getName() + " " + getClickMethodName(element) + "){");
        methodBuilder.append("\n}");
        // 创建OnClick方法
        mClass.add(mFactory.createMethodFromText(methodBuilder.toString(), mClass));
    }

    /**
     * 获取点击方法的名称
     */
    public String getClickMethodName(Element element) {
        String[] names = element.getId().split("_");
        // aaBbCc
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < names.length; i++) {
            if (i == 0) {
                sb.append(names[i]);
            } else {
                sb.append(Util.firstToUpperCase(names[i]));
            }
        }
        return sb.toString();
    }

    /**
     * 在加载布局后根据activity Fragement View 来初始化注解框架
     */
    private void generateInjects() {
        PsiClass activityClass = JavaPsiFacade.getInstance(mProject).findClass(
                "android.app.Activity", new EverythingGlobalScope(mProject));
        PsiClass fragmentClass = JavaPsiFacade.getInstance(mProject).findClass(
                "android.app.Fragment", new EverythingGlobalScope(mProject));
        PsiClass supportFragmentClass = JavaPsiFacade.getInstance(mProject).findClass(
                "android.support.v4.app.Fragment", new EverythingGlobalScope(mProject));

        // Check for Activity class
        if (activityClass != null && mClass.isInheritor(activityClass, true)) {
            generateActivityBind();
            // Check for Fragment class
        }
//        else if ((fragmentClass != null && mClass.isInheritor(fragmentClass, true)) || (supportFragmentClass != null && mClass.isInheritor(supportFragmentClass, true))) {
//            generateFragmentBindAndUnbind();
//        }
    }

    /**
     * activity在加载布局后生成ViewUtils.inject(this)代码
     */
    private void generateActivityBind() {
        PsiElementFactory mFactory = JavaPsiFacade.getElementFactory(mProject);
        if (mClass.findMethodsByName("onCreate", false).length == 0) {
            // Add an empty stub of onCreate()
            StringBuilder method = new StringBuilder();
            method.append("@Override protected void onCreate(android.os.Bundle savedInstanceState) {\n");
            method.append("super.onCreate(savedInstanceState);\n");
            method.append("\t// TODO: add setContentView(...) invocation\n");
            method.append(VIEW_BIND);
            method.append("(this);\n");
            method.append("}");

            mClass.add(mFactory.createMethodFromText(method.toString(), mClass));
        } else {
            PsiMethod onCreate = mClass.findMethodsByName("onCreate", false)[0];
            if (!containsViewInjectLine(onCreate, VIEW_BIND)) {
                for (PsiStatement statement : onCreate.getBody().getStatements()) {
                    // Search for setContentView()
                    if (statement.getFirstChild() instanceof PsiMethodCallExpression) {
                        PsiReferenceExpression methodExpression
                                = ((PsiMethodCallExpression) statement.getFirstChild())
                                .getMethodExpression();
                        // Insert ButterKnife.inject()/ButterKnife.bind() after setContentView()
                        if (methodExpression.getText().equals("setContentView")) {
                            onCreate.getBody().addAfter(mFactory.createStatementFromText(
                                    VIEW_BIND + "(this);", mClass), statement);
                            break;
                        }
                    }
                }
            }
        }
    }


    /**
     * 判断OnCreate中是否有初始化注解框架代码
     * @param method
     * @param line
     * @return
     */
    private boolean containsViewInjectLine(PsiMethod method, String line) {
        final PsiCodeBlock body = method.getBody();
        if (body == null) {
            return false;
        }
        PsiStatement[] statements = body.getStatements();
        for (PsiStatement psiStatement : statements) {
            String statementAsString = psiStatement.getText();
            if (psiStatement instanceof PsiExpressionStatement && (statementAsString.contains(line))) {
                return true;
            }
        }
        return false;
    }
```

我的CSDN博客：[https://blog.csdn.net/wenwins](https://blog.csdn.net/wenwins)  

我的简书博客：[https://www.jianshu.com/u/eb8a1db752af](https://www.jianshu.com/u/eb8a1db752af)
