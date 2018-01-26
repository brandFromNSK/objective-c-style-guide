Code-style-guide
======
# Conceptual

## Preconditions and verification

### Methods

```
- (void)cewlSelector:(id)something {
	if( 1 <= 0 ){
		[self doSomething];
		return;
	}
	// код
	if( 2 <= 1) {
		[self doSomething];
		return;
	}
	// Еще код
	for( int x = 0; x < 10; x++ ){
		if( x <= 4 ) {
			// ой, опять код
			return;
		}
		// код код код
	}
	[self doSomething];
}
```
over
```
- (void)cewlSelector:(id)something {
	if( 1 > 0 ){
		// код
		if( 2 > 1) {
			// Еще код
			for( int x = 0; x < 10; x++ ){
				//Все еще код
				if( 3 < 4 ) {
					// код код код
					return;
				}
				// ой, опять код
			}
		}
	}
	[self doSomething];
	return;
}
```
Мотивация: 

Вложенность зло. Причин несколько:

* Стиль. Идентация в 3-4-5 уровней выглядит не очень, особенно на ревью в двухпанельных режимах. Плюс любые правки по расширению логики (добавление нового условия), добавляют идентации.
* В этих ветвях реально легко запутаться. Разворот условий помогает идти по коду ближе к императивной модели, к которой, как ни крути все сводится, встречая выбросы, если они нужны.
* Все языки для новых белых людей начинают насаждать этот концепт. См. `guard else` 

### initialization

```
- (instancetype)init {
    self = [super init];
    return self;
}
```
over:
```
- (instancetype)init {
    self = [super init];
    if( nil == self ){
        return nil;
    }
    return self;
}
```
over:
```
- (instancetype)init 
    self = [super init];
    if( self ) {
        return self;
    }
    return nil;
}
```

Мотивация:

**Проверка на nil:** Если пошариться по Foundation/UIKit, можно обнаружить что почти все классы содержат в заголовке конструкцию `NS_ASSUME_NONNULL_BEGIN`. Почему она отсутствует у NSObject? Есть мысль что сделано это в целях обратной совместимости, потому что иначе вернуть `nil` из конструктора не представляется возможным, поскольку преобразование `nonnull -> nullable` конфликтно (`Warning: conflicting nullability specifier on return types, 'nullable' conflicts with existing specifier 'nonnull'`). Но, например `initWithCoder:` вполне может вернуть и `nil` если объект не удалось распаковать.

**Sic!**: `initWithCoder:` все еще `nullable` и сверка результата инициализации `super` -- обязательна

## Defensive flow

*Мотивация: написанный код никогда не будет использоваться как вы ожидаете, максимум что можно сделать, назапрещать это насколько можно.*

### Initialization
- `NS_DESIGNATED_INTIALIZER` и обязательное обьявление инициализатора в интерфейсе даже если это обычный `init`
- `NS_UNAVAILABLE` и обязательное обьявление инициализатора в интерфейсе если он не допустим в данном контексте, пример:
```
- (instancetype)initWithConfig:(id<SomeConfig>)config andValue:(CGFloat)valueThatCanSetByDefault NS_DESIGNATED_INITALIZER;
- (instancetype)initWithConfig:(id<SomeConfig>)config;
- (instancetype)init NS_UNAVAILABLE;
```

### Assertions and using warnings

#### Entry point

- `NSAssert` на все что не `nullable` по логике. Nullability markers - отдельный вопрос.

- `NSAssert` на все валидные значения, если скоуп ограничен.

- `NSAssert` на все запрещенные ветви кода

- `NSAssert` на все абстрактные методы назначенные к перегрузке

```
NSAssert( nil != contactId, @"Value of 'contactId' must never be nil");
NSAssert( contacts.count > 0, @"Count of 'contacts' must be greater or equal 1");
NSAssert( error.code != 300, @"Something went wrong. Double-check why error contains this code");
NSAssert(NO, @"Abstract method! Must be overridden in child class!");
```

#### Conditions

- `switch(x){default:break;}` под запретом, либо с ассертом

- `if-else` под запретом, либо с ассертом, либо с `if-elseif-else`

*Мотивация: необработанные значения виднее, и попадают в варнинги при компиляции, что допускает*

### Yoda-style conditions

`const == param` вместо `param == const`

*Мотивация: const = param не сработает в отличии от обратного. Не так частно, но больно если напороться.*

### Protocols declaration

`@optional` в протоколах запрещен. 

*Мотивация: Протокол это абстракция поведения, любое "необязательное" поведение нарушение ISP. Плюс, необходимость возиться с постоянным проверками на `respondsToSelector:`по вызоывам*

# Hygiene

### Code Blocks usage

- В селекторы передаем через переменную, если не последний параметр. Мотивация, не рвать полотно кода на длинных сигнатурах.

```
id animation = ^(id<UIViewControllerTransitionCoordinatorContext> context){
    // do some code
};
id completion = ^(id<UIViewControllerTransitionCoordinatorContext> context){
    // do some code
};
[self.transitionCoordinator animateAlongsideTransition:animation completion:completion];
```

- Количество кода в блоке подлежит обсуждению. Хорошая практика, особенно при передаче куда-либо, проксировать на какой либо метод

```
NSObject *__weak weakSelf = self;
operation.successHandler = ^(id response) {
    [weakSelf success:response];
};
```

### Comments

- Законменченый код и комментарии поясняющие чем положено заполнять метод - зло и должны вырезаться до попадания в VCS

- `TODO:`, `FIXME:` - обязательны для всех ситуаций если работа не закончена.

- Если это абстрактный метод с пустым поведением, рекомендуется оставить комментарий `// Abstract method, shoul be overriden`, если метод обязан быть перегружен см секцию Entry point

- Законменченый код допустим только с комментариями `TODO:`, `FIXME:` и приложением ссылки-тикета техдолга.

- javadoc на сложные методы

### Chaining methods

Чейнинг очень затрудняет отладку, очень затрудняет дифы и превращает код в неконтролируемые снаряды.
Поэтому чейнинг больше 3 уровней - допускается с комментариями. Чейнинг для параметров - запрещен впринципе.

```
UIWindow *keyWindow = [UIApplication sharedApplication].keyWindow;
UIView *rootVCView = keyWindow.rootViewController.view;
const CGFloat viewWidth = rootVCView.frame.size.width;

[self doSomething:viewWidth];
```
over 
```
[self doSomething:[UIApplication sharedApplication].keyWindow.rootViewController.view.frame.size.width];
```


# Code Style

### Statement brackets

```
if( y ){}
for( x = 1... ){}
while( true ){}
```
over 
```
if (y){}
for (x = 1...){}
while (true){}
```

*Мотивация: почему то никто не зовет функции `CGFloat (0.0f...) ;`, но считается нормальным провернуть такое же для оператора*

*Внутренняя отбивка скобок -- на совести разработчика. При усложении выражения, рекомендуется для функций в том числе.*

### Force typing

```
CGFloat x = 0.0f;
double y = 0.0;
NSInteger z = 1;
```

*Мотивация: при передаче или удлинении методов, намного очевидней что за тип данных у константы, если ее значение корректено*

*Наличие дробной части -- просто дань гигиене кода и необязательно* 

### Force boolean values

bools: `NO == bValue`, but `YES\NO == [self doSomething]`, but `bValue`

objects: `nil != \ == object`

*Мотивация:*

- Внезапно, но `!bValue` очень сложно заметить на фиксах в предрелизной суматохе

- Внезапно, но сравнение с типом результата метода - помогает понять что возвращается, не изучая сигнатуры

- Внезапно, но `nil ` и `NO` уже давно очень разные понятия. Первое это `__DARWIN_NULL`, который кастится по месту в `Nil` и `nil`, но не `NULL`, второе имеет тип `boolean`. Что происходит под `__DARWIN_NULL` неизвестно и неожиданно. Поэтому на касты закладываться - опасно.


### Attributes

~~`__weak` `__strong` -- относятся к указателю (`*`) и должны быть установлены после него. Запись `__weak NSObject *welf = self` некорректна, хотя и допускается компиляторами.~~ 

⋅⋅⋅Тут интересно. Ссылочка на стандарт : <http://clang.llvm.org/docs/Block-ABI-Apple.html#importing-attribute-nsobject-variables>

⋅⋅⋅Утверждается, и приводится в пример, что `__block __weak` указываются до литерала, что бы быть правильно развернутыми. Но, их сосед, `__attribute__((NSObject))`, по параметрам связки с блоком (`enum BLOCK_FIELD`) все таки указывается на положенном месте, сразу после указания указателя `*`. Почему так - непонятно. Однако, это стандарт.

- `__block` `__weak` `__strong` - указываются как префикс перед любым литералом.

- `__unused` -- накладывается на литерал, поэтому пишется после него. Для функций и методов - после закрывающей скобки параметров, перед открывающей фигурной скобкой.

## Properties

### Properties declaration

- Порядок: **atomic, strong, readonly, nullable** -- Принято читать сигнатуры переменных (в данном случае, пропертей) справа налево: *Имя, тип, может ли быть пустой, может ли перезаписываться, как держится, потокобезопасность.*

- access-modifiers - допустимо опускать если это readwrite проперти с хотя бы одним стандартным аксессором/сеттером

- nullability - на усмотрение в проекте

- copy over strong - обязательно для NSValue, NSNumber, NSString, blocks. Мутабельные объекты на совести разработчика. Остальное на усмотрение.

- Атомарность - В UIKit/Foundation всего пара мест где используется atomic, но тем не менее это значение по умолчанию. Так что обязательно.

### Properties usage

Все что проперти - должно использоваться как проперти.

Это же относится и к class-level пропертям

