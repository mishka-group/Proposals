
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

### ساخت یک پلاگین ساده
در اولین مرحله شما باید رویدادی که نیاز دارید را انتخاب کنید. این رویداد در اسناد مربوط به سیستم مدیریت محتوا میشکا ثبت می باشد. به عنوان مثال در این آموزش کوتاه رویداد `on_user_after_login` را انتخاب می کنیم. این رویداد در زمانی اجرا می شود که یک ورود مجاز به سیستم انجام گردد. لیست کل `event` های  هسته `cms` را می توانید [اینجا](https://github.com/mishka-group/mishka-cms/blob/master/apps/mishka_installer/lib/plugin_manager/event/event.ex) ببنید

در مرحله بعدی یک پکیج `heex` درست کنید داکیومنت آن را می توانید در [اینجا](https://hex.pm/docs/usage) ببنید. حال تصمیم با شماست که این پلاگین را فقط در گیت هاب خود داشته باشید یا آن را در `heex` نیز منتشر کنید. در مرحله بعد شما باید یک ماژول را به عنوان اکشن در نظر بگیرید ( در پکیجی که منتشر کردید)

در این آموزش به عنوان مثال یک ماژول با نام `MishkaUser.CorePlugin.Login.SuccessLogin` درست می کنیم در ابتدای این ماژو شما می توانید با ماکرو از پیش تعریف شده جلو بروید که پیشنهاد روی همین موضوع می باشد یا می توانید خودتان یک `Genserver` بسازید. ولی فرض بر اینکه از ماکرو `__using__` ماژول `Hook` استفاده می کنید. 

```elixir
defmodule MishkaUser.CorePlugin.Login.SuccessLogin do
  alias MishkaInstaller.Reference.OnUserAfterLogin
  use MishkaInstaller.Hook,
      module: __MODULE__,
      behaviour: OnUserAfterLogin,
      event: :on_user_after_login,
      initial: []
      
      .....
end
```
همانطور که می بنید در خط اول `callback` رویداد انتخاب شده را `alias` کردیم که این کار انتخابی می باشد ولی برای تمیز شدن فایل و کوتاه تر شدن لاین ها انجام گردید و در خط دوم ماژول `MishkaInstaller.Hook` ماکرو `__using__` صدا زده می شود و چهار پارامتر که اولی ماژولی که به عنوان اکشن پلاگین در نظر گرفته ای به آن داده می شود و دومی نیز `behaviour` رویداد مورد نظر  و سومی نیز `atom` یا نام یونیک رویداد و چهارمی نیز ورودی اولیه `Genserver` می باشد که بستگی به استراتژی شما دارد در این آموزش به صورت یک لیست خالی در نظر گرفته شده است.
> توجه کنید: ورودی به تابعی به نام `initial` ارسال می گردد و بر اساس انتخاب رویداد `OnUserAfterLogin` باید ورودی آن از نوع لیست باشد. و وقتی `Hook` در ماژول پلاگین شما `use` می شود باید دو تابع اصلی به نام ها `initial` و `call` اضافه شود


در تابع اول: در زمانی که سرور فونیکس شما بالا می آید با بانک اطلاعاتی شما چک می کند اگر داده ای برای پلاگین شما ثبت شده است پس از آن داده برای ساخت یک `state` داینامیک که سوپروایزر نیز می شود استفاده می کند و اگر نشده باشد از اطلاعات اولیه پلاگین شما به آن می دهید استفاده می کند و در دیتابیس نیز همان را ذخیره می کند.
> دلیل این کار این می باشد ممکن است شما از پارامتر `extra` که در استراکت `state` قرار داده شده است برای کانفیگ کاربری استفاده کنید و هر زمانی که سرور خاموش روشن می شود نباید این پارامتر فراموش شود .

```elixir
@spec initial(list()) :: {:ok, OnUserAfterLogin.ref(), list()}
def initial(args) do
  event = %PluginState{name: "MishkaUser.CorePlugin.Login.SuccessLogin", event: Atom.to_string(@ref), priority: 1}
  Hook.register(event: event)
  {:ok, @ref, args}
end
```
همانطور که می بنید ما یک پلاگین با استراکت از پیش ساخته شده
```elixir
 defstruct [:name, :event, priority: 1, status: :started, depend_type: :soft, depends: [], extra: []]
```

در پارامتر `event` ارزش گذاری کردیم و آن را در تابع `register` از ماژول `Hook` به سیستم ثبت نمودیم. و در آخر نیز `{:ok, @ref, args}` خروجی را بر اساس `callback` مورد نظر پس دادیم. در [اینجا](https://github.com/mishka-group/mishka-cms/blob/master/apps/mishka_installer/lib/plugin_manager/event/reference/on_user_after_login.ex) می توانید ببنید

```elixir
 @type reason() :: map() | String.t() # output of state for this event
 @callback initial(list()) :: {:ok, ref(), list()} | {:error, ref(), reason()} # Register hook
```
> باید توجه داشت هر پلاگین علاوه بر یک `state` که با ماژول های زیر سیستم `mishka_installer`  سوپروایز می شود دارد یک `state` به نام خود ماژول نیز دارد که از نوع یک `Genserver` ساده می باشد پس زمان آن رسیده است که در یکی از اپلیکیشن های زیر سیستم ها ثبت شود. حال این می تواند یک پروژه جداگانه در ساختار `DDD` پروژه میشکا باشد یا خواه می خواهد به صورت مثال در زیر سیستم  `mishka_user` باشد.
> در نسخه بعدی شرایط برای ثبت مجدد پلاگین ها در سیستم مدیریت محتوا بعد از خاموش و روشن شدن سرور نیز ایجاد می گردد
```elixir
defmodule MishkaUser.Application do
  @moduledoc false
  use Application
  @impl true
  def start(_type, _args) do
    children = [
      %{id: MishkaUser.CorePlugin.Login.SuccessLogin, start: {MishkaUser.CorePlugin.Login.SuccessLogin, :start_link, [[]]}}
    ]
    opts = [strategy: :one_for_one, name: MishkaUser.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

حال زمان آن رسیده است که دوباره به ماژول پلاگین خود برگردید و فانکشن `call` را بر اساس `callback` رویداد مورد نظر بسازید

```elixir
@spec call(OnUserAfterLogin.t()) :: {:reply, OnUserAfterLogin.t()}
def call(%OnUserAfterLogin{} = state) do
  YOUR CODE, OR Change state and pass it to Below Tuple
  {:reply, state}
end
```

همانطور که در اسناد سیستم مدیریت محتوا میشکا نیز ثبت شده است و همینطور در `callback` رویداد مورد نظر شما دو راه خواهید داشت می توانید از `{:reply, state}` استفاده کنید و `state` را به پلاگین های دیگر فعال در این رویداد انتقال بدهید یا می توانید همینجا از `{:reply, :halt, state}` استفاده کنید آخرین `state` چه ویرایش شده باشد چه نشده باشد را به `Hook` برگردانید. با این روش حلقه فراخوانی دیگر پلاگین ها شکسته می شود و تا همینجا بسنده می گردد.

حال زمان استارت مجدد سرور فونیکس و پروژه فرا رسیده است



### منابع:

1. https://github.com/mishka-group/mishka-cms/issues/148
2. https://youtu.be/7LX67RjkiGc
3. https://github.com/processone/ejabberd/blob/master/src/gen_mod.erl#L89-L92
4. https://hexdocs.pm/plug/Plug.Conn.html
5. https://www.ejabberd.im/
6. [How to add and install deps without stopping phoenix server (hex package)](https://elixirforum.com/t/how-to-add-and-install-deps-without-stopping-phoenix-server-hex-package/44888)
7. [How to create Ecto temporary tables for test a library](https://elixirforum.com/t/how-to-create-ecto-temporary-tables-for-test-a-library/44890)
8. [Need advice to implement custom extension and template installation in an elixir CMS](https://elixirforum.com/t/need-advice-to-implement-custom-extension-and-template-installation-in-an-elixir-cms/44863/)
9. [How to check a macro was called in a module](https://elixirforum.com/t/how-to-check-a-macro-was-called-in-a-module/44674)
10. https://docs.joomla.org/Plugin/Events
11. https://developer.wordpress.org/plugins/hooks/actions/
12. https://elixirforum.com/t/how-to-get-ecto-stream-respond-in-exunit/46138
13. https://elixirforum.com/t/genserver-doesnt-consider-ecto-sandbox/46105
14. https://elixirforum.com/t/dynamicsupervisor-terminate-genserver-state-in-transient-restart-method/45932/2
15. https://elixirforum.com/t/genserver-terminate-strategy-and-re-bind-data/45779
16. https://elixirforum.com/t/how-to-send-sandbox-allow-for-each-dynamic-supervisor-testing/46422/4
17. https://github.com/hexpm/hexpm/issues/1124