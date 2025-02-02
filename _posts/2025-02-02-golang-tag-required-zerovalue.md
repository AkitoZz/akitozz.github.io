---
layout: post
title: gin tag required 校验零值异常
categories: [golang, gin, tag, validator]
description: golang, gin, tag, 零值异常
keywords: golang, gin, tag, validator
---

## 问题现象

在使用gin框架定义post请求body的结构体的时候，定义了一个bool类型的参数，并且设置成了required参数，但是在postman发送模拟请求的时候，及时携带了该参数还是报参数异常错误。

代码

```golang

请求体定义：

type TaskStartReqBody struct {
    Date            string json:"date" binding:"required"
    EnableRankCheck bool   json:"enable_rank_check" binding:"required"
}

实际的post请求body：

{    
    "date": "2025-01",    
    "enable_rank_check": false
}

```

请求body包含必须参数但是报错：
```
Field validation for 'EnableRankCheck' failed on the 'required' tag
```

但如果请求体为以下，则并不会报错
```
{    
    "date": "2025-01",    
    "enable_rank_check": true
}
```

## 错误原因

首先我们先看一下源码(go 1.16, validator v8)的调用关系，参数校验是从gin框架提供的`ShouldBindJSON`接口进入的，里面是调用 `binging`包里的
json.go的bind函数
```
func (jsonBinding) Bind(req *http.Request, obj interface{}) error {
	if req == nil || req.Body == nil {
		return fmt.Errorf("invalid request")
	}
	return decodeJSON(req.Body, obj)
}

func (jsonBinding) BindBody(body []byte, obj interface{}) error {
	return decodeJSON(bytes.NewReader(body), obj)
}

func decodeJSON(r io.Reader, obj interface{}) error {
	decoder := json.NewDecoder(r)
	if EnableDecoderUseNumber {
		decoder.UseNumber()
	}
	if err := decoder.Decode(obj); err != nil {
		return err
	}
	return validate(obj)
}

```


decoder.Decode函数是将请求体绑定到结构体中，validate是真正的校验逻辑入口。

而validator会调用ValidateStruct函数，里面的ValidateStruct函数中的v.validate.Struct会引用validator包中的校验函数实现。

```
func (v *defaultValidator) ValidateStruct(obj interface{}) error {
	value := reflect.ValueOf(obj)
	valueType := value.Kind()
	if valueType == reflect.Ptr {
		valueType = value.Elem().Kind()
	}
	if valueType == reflect.Struct {
		v.lazyinit()
		if err := v.validate.Struct(obj); err != nil {
			return err
		}
	}
	return nil
}
```

```
Struct -> ensureValidStruct -> tranverseStruct

其中比较核心的逻辑是这里

if first || ct == nil || ct.typeof != typeStructOnly    {

	    for _, f := range cs.fields {

			if partial {

				_, ok = includeExclude[errPrefix+f.Name]

				if (ok && exclude) || (!ok && !exclude) {
					continue
				}
			}

			v.traverseField(topStruct, currentStruct, current.Field(f.Idx), errPrefix, nsPrefix, errs, partial, exclude, includeExclude, cs, f, f.cTags)
		}
	}

	// check if any struct level validations, after all field validations already checked.
	if cs.fn != nil {
		cs.fn(v, &StructLevel{v: v, TopStruct: topStruct, CurrentStruct: current, errPrefix: errPrefix, nsPrefix: nsPrefix, errs: errs})
	}


也就是要校验的如果是字段，就调用traverseField函数继续检查，要是校验结构体则直接使用cs.fn进行校验。注意这个cs.fn，fn是函数指针也就是校验字段绑定的校验函数。
```

traverseField 函数是整个validator包校验的核心，里面需要关注的函数就是普通流程处理中的这里：
```
default:
			if !ct.fn(v, topStruct, currentStruct, current, typ, kind, ct.param) {

				ns := errPrefix + cf.Name

				errs[ns] = &FieldError{
					FieldNamespace: ns,
					NameNamespace:  nsPrefix + cf.AltName,
					Name:           cf.AltName,
					Field:          cf.Name,
					Tag:            ct.aliasTag,
					ActualTag:      ct.tag,
					Value:          current.Interface(),
					Param:          ct.param,
					Type:           typ,
					Kind:           kind,
				}

				return

			}

			ct = ct.next
		}
```

关于ct也就是tag，tag的函数指针fn是怎么绑定的，我们可以看到初始化New函数中
```
func New(config *Config) *Validate {

	tc := new(tagCache)
	tc.m.Store(make(map[string]*cTag))

	sc := new(structCache)
	sc.m.Store(make(map[reflect.Type]*cStruct))

	v := &Validate{
		tagName:      config.TagName,
		fieldNameTag: config.FieldNameTag,
		tagCache:     tc,
		structCache:  sc,
		errsPool: &sync.Pool{New: func() interface{} {
			return ValidationErrors{}
		}}}

	if len(v.aliasValidators) == 0 {
		// must copy alias validators for separate validations to be used in each validator instance
		v.aliasValidators = map[string]string{}
		for k, val := range bakedInAliasValidators {
			v.RegisterAliasValidation(k, val)
		}
	}

	if len(v.validationFuncs) == 0 {
		// must copy validators for separate validations to be used in each instance
		v.validationFuncs = map[string]Func{}
		for k, val := range bakedInValidators {
			v.RegisterValidation(k, val)
		}
	}

	return v
}

RegisterValidation是注册函数，而bakedInValidators就是注册的对象，里面就有
var bakedInValidators = map[string]Func{
	"required":     HasValue,
}
```


也就是说required tag的校验需要看这个HasValue函数。

```
func HasValue(v *Validate, topStruct reflect.Value, currentStructOrField reflect.Value, field reflect.Value, fieldType reflect.Type, fieldKind reflect.Kind, param string) bool {

	switch fieldKind {
	case reflect.Slice, reflect.Map, reflect.Ptr, reflect.Interface, reflect.Chan, reflect.Func:
		return !field.IsNil()
	default:
		return field.IsValid() && field.Interface() != reflect.Zero(fieldType).Interface()
	}
}
```

看到源码的这里也就破案了，对于非 切片，map，指针，接口，chan，函数的类型，如果值是该类型的零值那么就会判断为不存在，所以上述使用情况的bool值，传递进来的是false的时候，会因为等于自己类型的零值而导致`required`判断失败。

## 解决方法

看到上述源码的判断逻辑后，解决方法也就呼之欲出啦，把`bool`类型定义换成`*bool`也就是指针类型，这样就会判断是否为空指针来确认是否满足`required`了，不过在取值的时候也注意一下用指针获取传递进来的bool值就好了。

## 后记

刚遇到这个问题的时候我第一反应是自己的单词拼写错误导致参数没传对，确认没问题后网上看到也有遇到相同问题的，有让使用`omitempty`运行空值绕开的，也有直接说定义为指针的，不过都没说明底层原因。看了下源码后一下思路就清晰了。
