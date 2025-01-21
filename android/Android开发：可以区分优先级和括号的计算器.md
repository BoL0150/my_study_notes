# Android开发：可以区分优先级和括号的计算器

此计算器是基于将中缀表达式转化为后缀表达式，从而对表达式求值的算法。

成果图如下：

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210413035451331.png" alt="image-20210413035451331" style="zoom: 67%;" />

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210413034203162.png" alt="image-20210413034203162" style="zoom: 67%;" />

## xml布局文件

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
        xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:app="http://schemas.android.com/apk/res-auto"
        xmlns:tools="http://schemas.android.com/tools"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical"
        android:paddingBottom="10dp"
        android:paddingLeft="10dp"
        android:paddingRight="10dp"
        android:paddingTop="10dp"
        tools:context="com.example.caculator.MainActivity"
        >

    <EditText
            android:id="@+id/et_input"
            android:layout_width="match_parent"
            android:layout_height="60dp"
            android:inputType="none"
            android:gravity="center|right"
            android:background="#f0f0f0"/>

    <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginTop="20dp"
            android:orientation="horizontal">

        <Button
                android:id="@+id/left_parenthese"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="("
                android:textSize="20sp"/>

        <Button
                android:id="@+id/right_parenthese"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginLeft="5dp"
                android:text=")"
                android:textSize="20sp" />

    </LinearLayout>

    <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginTop="10dp"
            android:orientation="horizontal">

        <Button
                android:id="@+id/btn_clear"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="C"
                android:textSize="20sp"/>

        <Button
                android:id="@+id/btn_del"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginLeft="5dp"
                android:text="DEL"
                android:textSize="20sp"/>

        <Button
                android:id="@+id/btn_divide"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginLeft="5dp"
                android:text="/"
                android:textSize="20sp"/>

        <Button
                android:id="@+id/btn_multply"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginLeft="5dp"
                android:text="*"
                android:textSize="20sp"/>

    </LinearLayout>

    <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginTop="10dp"
            android:orientation="horizontal">

        <Button
                android:id="@+id/btn_7"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="7"
                android:textSize="20sp"/>

        <Button
                android:id="@+id/btn_8"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginLeft="5dp"
                android:text="8"
                android:textSize="20sp"/>

        <Button
                android:id="@+id/btn_9"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginLeft="5dp"
                android:text="9"
                android:textSize="20sp"/>

        <Button
                android:id="@+id/btn_minus"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginLeft="5dp"
                android:text="-"
                android:textSize="20sp"/>

    </LinearLayout>

    <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginTop="10dp"
            android:orientation="horizontal">

        <Button
                android:id="@+id/btn_4"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="4"
                android:textSize="20sp"/>

        <Button
                android:id="@+id/btn_5"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginLeft="5dp"
                android:text="5"
                android:textSize="20sp"/>

        <Button
                android:id="@+id/btn_6"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginLeft="5dp"
                android:text="6"
                android:textSize="20sp"/>

        <Button
                android:id="@+id/btn_plus"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginLeft="5dp"
                android:text="+"
                android:textSize="20sp"/>

    </LinearLayout>

    <LinearLayout
            android:layout_width="fill_parent"
            android:layout_height="wrap_content"
            android:layout_marginTop="10dp"
            android:orientation="horizontal">

        <LinearLayout
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:orientation="vertical">

            <LinearLayout
                    android:layout_width="wrap_content"
                    android:layout_height="wrap_content"
                    android:orientation="horizontal">

                <Button
                        android:id="@+id/btn_1"
                        android:layout_width="wrap_content"
                        android:layout_height="wrap_content"
                        android:text="1"
                        android:textSize="20sp"/>

                <Button
                        android:id="@+id/btn_2"
                        android:layout_width="wrap_content"
                        android:layout_height="wrap_content"
                        android:layout_marginLeft="5dp"
                        android:text="2"
                        android:textSize="20sp"/>

                <Button
                        android:id="@+id/btn_3"
                        android:layout_width="wrap_content"
                        android:layout_height="wrap_content"
                        android:layout_marginLeft="5dp"
                        android:text="3"
                        android:textSize="20sp"/>

            </LinearLayout>

            <LinearLayout
                    android:layout_width="wrap_content"
                    android:layout_height="wrap_content"
                    android:paddingTop="10dp"
                    android:weightSum="1"
                    android:orientation="horizontal">

                <Button
                        android:id="@+id/btn_0"
                        android:layout_width="180dp"
                        android:layout_height="wrap_content"
                        android:text="0"
                        android:textSize="20sp"/>

                <Button
                        android:id="@+id/btn_point"
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:layout_marginLeft="5dp"
                        android:layout_weight="18.10"
                        android:text="."
                        android:textSize="20sp"/>


            </LinearLayout>

        </LinearLayout>

        <Button
                android:id="@+id/btn_equal"
                android:layout_width="wrap_content"
                android:layout_height="match_parent"
                android:layout_marginLeft="5dp"
                android:background="@android:color/holo_orange_light"
                android:text="="
                android:textSize="20sp"/>

    </LinearLayout>

</LinearLayout>
```

## Activity

- 定义监听器类，注册监听器对象

  ```java
  //基于监听接口的事件处理机制。第一步：定义监听器类，在监听器中针对事件编写响应的处理代码
  public class MainActivity extends AppCompatActivity implements View.OnClickListener {
      private Button btn_0;//0数字按钮
      private Button btn_1;//1数字按钮
      private Button btn_2;//2数字按钮
      private Button btn_3;//3数字按钮
      private Button btn_4;//4数字按钮
      private Button btn_5;//5数字按钮
      private Button btn_6;//6数字按钮
      private Button btn_7;//7数字按钮
      private Button btn_8;//8数字按钮
      private Button btn_9;//9数字按钮
      private Button btn_point;//小数点按钮
      private Button btn_clear;//clear按钮
      private Button btn_del;//del按钮
      private Button btn_plus;//+按钮
      private Button btn_minus;//-按钮
      private Button btn_multply;//*按钮
      private Button btn_divide;//除号按钮
      private Button btn_equal;//=按钮
  
      private Button left_parenthese;
      private Button right_parenthese;
      private EditText editText;
  
      @Override
      protected void onCreate(Bundle savedInstanceState) {
          super.onCreate(savedInstanceState);
          setContentView(R.layout.activity_main);
          btn_0 = (Button) findViewById(R.id.btn_0);
          btn_1 = (Button) findViewById(R.id.btn_1);
          btn_2 = (Button) findViewById(R.id.btn_2);
          btn_3 = (Button) findViewById(R.id.btn_3);
          btn_4 = (Button) findViewById(R.id.btn_4);
          btn_5 = (Button) findViewById(R.id.btn_5);
          btn_6 = (Button) findViewById(R.id.btn_6);
          btn_7 = (Button) findViewById(R.id.btn_7);
          btn_8 = (Button) findViewById(R.id.btn_8);
          btn_9 = (Button) findViewById(R.id.btn_9);
          btn_point = (Button) findViewById(R.id.btn_point);
          btn_clear = (Button) findViewById(R.id.btn_clear);
          btn_del = (Button) findViewById(R.id.btn_del);
          btn_plus = (Button) findViewById(R.id.btn_plus);
          btn_minus = (Button) findViewById(R.id.btn_minus);
          btn_multply = (Button) findViewById(R.id.btn_multply);
          btn_divide = (Button) findViewById(R.id.btn_divide);
          btn_equal = (Button) findViewById(R.id.btn_equal);
          editText = (EditText) findViewById(R.id.et_input);
          left_parenthese=(Button)findViewById(R.id.left_parenthese);
          right_parenthese=(Button)findViewById(R.id.right_parenthese);
          //注册监听器对象，将监听器对象绑定在该事件源上，当事件发生时，调用监听器中的事件响应代码
          btn_0.setOnClickListener(this);
          btn_1.setOnClickListener(this);
          btn_2.setOnClickListener(this);
          btn_3.setOnClickListener(this);
          btn_4.setOnClickListener(this);
          btn_5.setOnClickListener(this);
          btn_6.setOnClickListener(this);
          btn_7.setOnClickListener(this);
          btn_8.setOnClickListener(this);
          btn_9.setOnClickListener(this);
          btn_point.setOnClickListener(this);
          btn_clear.setOnClickListener(this);
          btn_del.setOnClickListener(this);
          btn_plus.setOnClickListener(this);
          btn_minus.setOnClickListener(this);
          btn_multply.setOnClickListener(this);
          btn_divide.setOnClickListener(this);
          btn_equal.setOnClickListener(this);
          left_parenthese.setOnClickListener(this);
          right_parenthese.setOnClickListener(this);
      }
  
  ```

- 处理事件：将点击按钮的结果显示在editText中

  ```java
  public Stack<Character> parStack=new Stack<>();
      @Override
      public void onClick(View view) {
          //view是事件源
          String editTextContent = editText.getText().toString();
          switch (view.getId()) {
              //只有当前面是空或者是运算符的时候才能加左括号
              case R.id.left_parenthese:
                  if (editTextContent.equals("")){
                      parStack.push('(');
                      editText.setText(editTextContent + "(");
                      break;
                  }
                  char previousChar=editTextContent.charAt(editTextContent.length()-1);
                  if (previousChar=='+'||previousChar=='-'||previousChar=='*'||previousChar=='/'){
                      parStack.push('(');
                      editText.setText(editTextContent + "(");
                  }
                  break;
              //只有当栈顶是左括号并且前面是数字时才能加右括号
              case R.id.right_parenthese:
                  if (editTextContent.equals(""))
                      break;
                  char previousChar2=editTextContent.charAt(editTextContent.length()-1);
                  if (parStack.isEmpty()||previousChar2=='+'||previousChar2=='-'||previousChar2=='*'||previousChar2=='/')
                      break;
                  parStack.pop();
                  editText.setText(editTextContent + ")");
                  break;
              case R.id.btn_0:
              case R.id.btn_1:
              case R.id.btn_2:
              case R.id.btn_3:
              case R.id.btn_4:
              case R.id.btn_5:
              case R.id.btn_6:
              case R.id.btn_7:
              case R.id.btn_8:
              case R.id.btn_9:
              case R.id.btn_point:
                  editText.setText(editTextContent + ((Button) view).getText());
                  break;
              case R.id.btn_plus:
              case R.id.btn_minus:
              case R.id.btn_multply:
              case R.id.btn_divide:
                  if (!editTextContent.equals(""))
                      editText.setText(editTextContent + ((Button) view).getText());
                  break;
              case R.id.btn_clear:
                  editTextContent = "";
                  editText.setText("");
                  break;
              case R.id.btn_del:
                  if (!editTextContent.equals("")) {
                      editTextContent = editTextContent.substring(0, editTextContent.length() - 1);
                      editText.setText(editTextContent);
                  }
                  break;
              case R.id.btn_equal:
                  if (!editTextContent.equals(""))
                      getResult();
                  break;
          }
  
      }
  ```

- 求出editText中表达式的结果：将表达式由中缀表达式转化为后缀表达式，再求解

  ```java
  public Stack<Double>numberStack=new Stack<>();
      public Stack<Character>operatorStack=new Stack<>();
      //运算结果
      private void getResult() {
          String exp = editText.getText().toString();
  
          String number="";
          for (int i=0;i<exp.length();i++){
              switch (exp.charAt(i)){
                  //如果是左括号，不管栈顶的优先级多高，直接进栈
                  case '(':
                      if (number!="") {
                          numberStack.push(Double.parseDouble(number));
                          number = "";
                      }
  
                      operatorStack.push('(');
                      break;
                  //如果是右括号，将栈中直到左括号为止（包括左括号）的所有符号全部出栈
                  case ')':
                      if (number!="") {
                          numberStack.push(Double.parseDouble(number));
                          number = "";
                      }
                      char c;
                      while ((c=operatorStack.pop())!='('){
                          numberStack.push(caculateTwoNumber(numberStack.pop(),numberStack.pop(),c));
                      }
                      break;
  
                  case '+':
                  case '-':
                      if (number!="") {
                          numberStack.push(Double.parseDouble(number));
                          number = "";
                      }
                      //如果符号栈不为空，由于+-优先级最低，所以需要将栈中的符号一直出栈，直到栈顶为左括号，或者栈为空
                      while (!operatorStack.isEmpty()&&operatorStack.peek()!='('){
                          numberStack.push(caculateTwoNumber(numberStack.pop(),numberStack.pop(),operatorStack.pop()));
                      }
                      operatorStack.push(exp.charAt(i));
                      break;
  
                  case '*':
                  case '/':
                      if (number!="") {
                          numberStack.push(Double.parseDouble(number));
                          number = "";
                      }
                      //优先级比自己高的全部出栈，直到栈为空
                      while (!operatorStack.isEmpty()&&(operatorStack.peek()=='*'||operatorStack.peek()=='/'))
                          numberStack.push(caculateTwoNumber(numberStack.pop(),numberStack.pop(),operatorStack.pop()));
                      operatorStack.push(exp.charAt(i));
                      break;
                  //如果是数字的话，先将数值的每一位合起来，当遇到符号位时，就形成了一个完整的数，再将这个数加入数值栈
                  default:
                      number+=exp.charAt(i);
              }
          }
          numberStack.push(Double.parseDouble(number));
          while (numberStack.size()>1&&!operatorStack.isEmpty()){
              numberStack.push(caculateTwoNumber(numberStack.pop(),numberStack.pop(),operatorStack.pop()));
          }
          editText.setText(String.valueOf(numberStack.pop()));
      }
      private double caculateTwoNumber(double right,double left,char operator){
          switch (operator){
              case '+':
                  return left+right;
              case '-':
                  return left-right;
              case '*':
                  return left*right;
              case '/':
                  return left/right;
              default:
                  return 0.0;
          }
      }
  ```

  

