
# مدیریت سیستم پلاگین ها در سیستم مدیریت محتوای میشکا

---
## هدف
ایجاد محیط مناسب و ساختار مورد انتظار برای توسعه سیستم مدیریت محتوا بدون تغییر کد هسته، به روز رسانی و همینطور توسعه امکانات کاربری. این بخش از سیستم میشکا به صورت یک زیر پروژه می باشد که `mishka_installer` نام گرفته است. 

## راه حل
مدیریت پلاگین در حقیقت یک `hook` برای کار کردن با `state` و بانک اطلاعاتی می باشد. به این صورت که شما یک پلاگین بر اساس  `Behavior` از قبل درست شده در هنگام شروع یا استارت سرور روی رم ثبت می کنید و این پروسه در بکگراند در دیتابیس نیز نسخه ای برای بکاپ نگهداری می شود.
> لازم به ذکر است در بار دوم اگر از قبل نسخه ای که از پلاگین درسته شده به وسیله برنامه نویس در دیتابیس وجود داشته باشد این بار اطلاعات از بانک اطلاعاتی روی رم ذخیره‌سازی می گردد وکل پروژه به دو بخش بزرگ از جمله: `مدیریت داده ها در استیت` و `توابع هوک برای برنامه نویس` و `رویداد ها`  تقسیم می شود.

### رویداد ها و مثال پلاگین ساده
---
رویداد ها در حقیقت اکشن ها و همینطور `Behavior` است که در هر بخش از سیستم مدیریت محتوا بر اساس نیاز قرار گرفته است. به عنوان مثال اگر کاربری ثبت نام کرد و ثبت نام آن موفقیت آمیز بود قبل از نمایش خروجی آخر این رویداد در آنجا اجرا می شود. پس به واسطه همین امکان برنامه نویس می تواند برای این رویداد به صورت نامحدود پلاگین های کاربری یا مدیریتی ایجاد کند. برای این کار می توانیم به صورت مثال بعد از ثبت نام کد فعال سازی حساب کاربری را پیامک کنیم یا همینطور می توانید پلاگینی برای ارسال ایمیل در این بخش فعال کنیم.

> لازم به ذکر است هر رویداد شامل چندین `callback` می باشد که خروجی و ورودی مورد انتظار را از برنامه نویس خواهن می باشد. و همینطور باید در شروع هر پلاگین آن را به عنوان `Behavior` در ماژول خود صدا بزند

#### به عنوان مثال:

```elixir
defmodule MishkaInstaller.Reference.OnUserAfterDeleteRole do
  @moduledoc """
    This event is triggered whenever a user's role is successfully deleted. if there is any active module in this section on state,
    this module sends a request as a Task tool to the developer call function that includes `user_info()`, `ip()`, `endpoint()`, `modifier_user()`.
    It should be noted; This process does not interfere with the main operation of the system.
    It is just a sender and is active for both side endpoints.
  """
  defstruct [:role_id, :ip, :endpoint, :conn]

  @type role_id() :: <<_::288>>
  @type user_id() :: role_id()
  @type conn() :: Plug.Conn.t() | Phoenix.LiveView.Socket.t()
  @type ip() :: String.t() | tuple() # User's IP from both side endpoints connections
  @type endpoint() :: :html | :api # API, HTML
  @type ref() :: :on_user_after_delete_role # Name of this event
  @type reason() :: map() | String.t() # output of state for this event
  @type registerd_info() :: MishkaInstaller.PluginState.t() # information about this plugin on state which was saved
  @type state() :: %__MODULE__{role_id: role_id(), ip: ip(), endpoint: endpoint(), conn: conn()}
  @type t :: state() # help developers to keep elixir style
  @type optional_callbacks :: {:ok, ref(), registerd_info()} | {:error, ref(), reason()}

  @callback initial(list()) :: {:ok, ref(), list()} | {:error, ref(), reason()} # Register hook
  @callback call(state()) :: {:reply, state()} | {:reply, :halt, state()}  # Developer should decide what and Hook call function
  @callback stop(registerd_info()) :: optional_callbacks() # Stop of hook module
  @callback restart(registerd_info()) :: optional_callbacks() # Restart of hook module
  @callback start(registerd_info()) :: optional_callbacks() # Start of hook module
  @callback delete(registerd_info()) :: optional_callbacks() # Delete of hook module
  @callback unregister(registerd_info()) :: optional_callbacks() # Unregister of hook module
  @optional_callbacks stop: 1, restart: 1, start: 1, delete: 1, unregister: 1 # Developer can use this callbacks if he/she needs
end
```

در طراحی این بخش سعی شده است در بیشتر `event` ها شبی به هم باشند جز مواردی که باید تغییر کند. پس ساختار کلی رویداد ها به این صورت می باشد. همانطور که در کد های بالا می بنید دو تابع `initial` و `call`ضروری می باشند و به همین شکل در کد ها نیز فراخوانی می گرند. این ساختار حتی در رویداد های سفارشی که بعدا به وسیله برنامه نویسان در زیر سیستم های دیگر پیاده‌سازی می شود نیز باید قرار بگیرد.
> ماکرو `__using__` در ماژول `hook` قرار گرفته است به صورت پیشفرض دو تابع  `initial` و `call` را صدا می زند و در صورت نبود آن ها `state` به صورت تغییر نکرده در بخشی که پلاگین های یک `event` صدا زده می شود فراخوانی می گردد تا سیستم `down` نشود.



### مدیریت `state` 
---
شما در این بخش این امکان را خواهید داشت که برای هر پلاگین یک `state` با نام پلاگین به صورت یکتا درست کنید و هر `state` نیز به صورت داینامیک سوپروایز می شود و شامل استراتژی های خطا و کنترل خطا نیز می باشد. به صورت مثال به هر صورتی دچار خطا شود ۴ بار تلاش می کند از بانک اطلاعاتی و دیگر بخش ها اطلاعات ضروری خودش را فراخوانی کند. هر پلاگین برای `push` شدن روی `state` نیاز دارد بر اساس استراکت تعریف شده عمل کند
```elixir
 defstruct [:name, :event, priority: 1, status: :started, depend_type: :soft, depends: [], extra: []]
```

و همینطور `typespec` های پارامترر های بالا به صورت زیر میم باشد

```elixir
  @type params() :: map()
  @type id() :: String.t()
  @type module_name() :: String.t()
  @type event_name() :: String.t()
  @type plugin() :: %PluginState{
    name: module_name(),
    event: event_name(),
    priority: integer(),
    status: :started | :stopped | :restarted,
    depend_type: :soft | :hard,
    depends: list(String.t()),
    extra: list(map())
  }
  @type t :: plugin()
```
> باید به این نکته توجه شود ممکن است این بخش در آینده شامل تغییر شود. این تغییرات بر اساس هدفی که دارند ورژن بندی خواهند شد فعلنه این ساختار برای نسخه ۰.۰.۱ سیستم مدیریت پلاگین ها می باشد

#### توابع این بخش به شرح زیر می باشد

1. ارسال داده با `push`: این تابع  داده مورد نظر شما را روی `state` و همینطور بانک اطلاعاتی ثبت می کند ولی منتظر پاسخ نمی ماند ممکن است خطا بدهد یا خیر 
2. ارسال داده با `push_call`: این تابع مثل `push` کار می کند با این تفاوت که  شما منتظر پاسخ می مانید
3. دریافت داده با `get` و `get_all`: این دو تابع همانطور که از اسم آن ها مشخص می باشد اطلاعات مربوط هر پلاگین یا پلاگین های یک رویداد خاص را از `state` استخراج می کند و همینطور می تواند بدون محدودیت از کل رم نیز اطلاعات تمام پلاگین ها را نیز فراخوانی کند.
4. حذف یک پلاگین از یک رویداد خاص یا حدف تمام پلاگین های یک رویداد خاص با تابع `delete`: این اطلاعات فقط از رم حذف می شود و تغییری روی بانک اطلاعاتی نخواهد داد
5. تابع تغییر وضعیت پلاگین به متوقف شده با تابع `stop` این تابع تغییر در بانک اطلاعاتی نیز اعمال می کند و فقط وضعیت یک پلاگین را تغییر می دهد
> این فایل شامل دیگر توابع نیز می شود از جمله استراتژی هایی که در هنگام خطا یا متوقف کردن یک پلاگین فراخوانی می شود می توانید در [اینجا](https://github.com/mishka-group/mishka-cms/blob/master/apps/mishka_installer/lib/plugin_manager/state/plugin_state.ex) ببنید


### توابع و ماژول `Hook`
---
در حقیقت این ماژول به صورت مستقیم در بخش هسته سیستم مدیریت محتوا و همینطور در پلاگین های ساخته شده به وسیله توسعه دهندگان استفاده می شود. هر پلاگین در شروع خودش ماکرو `__using__` از ماژول `Hook` را فراخوانی می کند و یک سری اطلاعات اولیه به آن داده و در حقیقت آن را بایند می کند. مثال:
```elixir
  alias MsihkaSendingEmailPlugin.TestEvent
  use MishkaInstaller.Hook,
      module: __MODULE__,
      behaviour: TestEvent,
      event: :on_test_event,
      initial: []
```
همانطور که می بنید در اولین خط رویداد مورد نظری که می خواهیم برای آن یک پلاگین بنویسیم را `alias` کردیم که انتخابی می باشد و شما می توانید ماژول را کامل فراخوانی کنید یا امکان `alias` را بزارید چون قرار است از آن در ادامه زیاد استفاده کنید و خطوط را طولانی می کند و در خط بعدی `use` انجام شده و ۴ پارامتر بعدی را ارزش گزاری می کنیم.

حال نیاز استدو تابع اصلی که قبلا به آن اشاره کردیم را فراخوانی کنیم
---
```elixir
def initial(args) do
  Logger.info("SendingEmail plugin was started")
  event = %PluginState{name: "MsihkaSendingEmailPlugin.SendingEmail", event: Atom.to_string(@ref), priority: 100}
  Hook.register(event: event)
  {:ok, @ref, args}
end

def call(%TestEvent{} = state) do
  {:reply, state}
end
````
اگر توجه کنید بر اساس `callback` هایی که قبلا برای رویداد مورد نظر درست شده اند ورودی و خروجی این دو تابع باید مواردی باشد که انتظار می رود وگرنه شما حتما در ادامه کار خطا خواهید داشت
> باید توجه کرد شما می توانید در تابع `call` کد های خود را بنویسید و `state` را تغییر بدهید. دقیقا محلی است که شما پلاگین خود را می نویسید حتی این قدرت به شما داده شده است به عنوان توسعه دهنده که اگر نیاز بود در همین فانکشن `state` را به برنامه برگردانید و دیگر پلاگین ها ثبت شده برای این رویداد را فراخوانی نکنید.

---

> باید توجه کنید پلاگین ها برای یک رویداد خاص بر اساس { اولویت و نام } مرتب می شوند و از کوچک به بزرگ به آن ها `state` اولیه داده می شود و هر پلاگین `state` را به دیگر پلاگین ها می دهد.

---

> در ماژول `hook` با برای تابع `call` نوع دیگری داریم که `state` را به روال قبل به پلاگین ها می دهد ولی در برنامه `state` اولیه را مصرف می کند ولی این امکان به برنامه نویس داده می شود که با `state` کار کند و موارد مربوط به خودش را به سیستم اضافه نماید شما می توانید ماژول hook و توابع آن را در [اینجا](https://github.com/mishka-group/mishka-cms/blob/master/apps/mishka_installer/lib/plugin_manager/event/hook.ex) ببنید


### منابع:

1. https://github.com/mishka-group/mishka-cms/issues/148
2. https://youtu.be/7LX67RjkiGc