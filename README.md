# claujson2-document
# first check!
    01. 64bit only
    02. has my own memory allocator.
    03. C++14~ 
    04. experimental!
    05. Array has my_own_vector<_Value>
    06. Object has my_own_vector<Pair<_Value, _Value>> (not a std::map!)
    07. in Object, key can be dupplicated, are not sorted, just in order of input.
    08. scanning - modified? simdjson, parsing - parallel 
    09. it is not read-only!
    10. _Value <- json_Value, (in destructor, no remove data(Array or Object!), no copy, only move or clone!
    11. Value <- wrap _Value, (in destructor, remove data(Array or Object!)
    
# claujson 라이브러리 문서 - using claude

> C++17 기반의 고성능 멀티스레드 JSON 파서 및 직렬화 라이브러리

---

## 목차

1. [개요](#개요)
2. [아키텍처 구조](#아키텍처-구조)
3. [핵심 클래스](#핵심-클래스)
   - [_Value](#_value)
   - [Array](#array)
   - [Object](#object)
   - [PartialJson](#partialjson)
   - [StructuredPtr](#structuredptr)
   - [Document](#document)
   - [String](#string)
   - [Arena (메모리 풀)](#arena-메모리-풀)
4. [파서 및 직렬화](#파서-및-직렬화)
   - [parser](#parser)
   - [writer](#writer)
5. [유틸리티](#유틸리티)
   - [diff / patch](#diff--patch)
   - [문자열 변환 함수](#문자열-변환-함수)
   - [Log 시스템](#log-시스템)
6. [타입 시스템](#타입-시스템)
7. [메모리 관리](#메모리-관리)
8. [사용 예시](#사용-예시)
9. [내부 구현 상세](#내부-구현-상세)

---

## 개요

claujson은 simdjson을 기반으로 구축된 C++ JSON 라이브러리입니다. 주요 특징:

- **멀티스레드 파싱**: JSON을 여러 청크로 분할하여 병렬 파싱 후 병합
- **Arena 기반 메모리 관리**: 별도 메모리 풀을 사용하여 할당/해제 오버헤드 최소화
- **이동 전용(Move-only) 설계**: `_Value`, `Value` 등의 핵심 타입은 복사 불가, 이동만 가능
- **Short String 최적화**: 11바이트 이하의 문자열은 힙 할당 없이 인라인 저장
- **병렬 직렬화**: JSON 트리를 분할하여 멀티스레드로 동시 직렬화

---

## 아키텍처 구조

```
claujson
├── claujson_internal.h    # 기반 타입, Arena, my_vector, Log 등
├── claujson_string.h      # String 클래스 (SSO 포함)
├── claujson.h             # _Value, Value, Document, StructuredPtr, parser, writer
├── claujson_array.h/.cpp  # Array 클래스
├── claujson_object.h/.cpp # Object 클래스
├── claujson_partialjson.h/.cpp # PartialJson (파싱 내부용)
└── claujson.cpp           # 파싱/직렬화 핵심 구현, diff/patch
```

### 의존 관계

```
_Value
 ├── Array*      (_array_ptr)
 ├── Object*     (_obj_ptr)
 ├── PartialJson* (_pj_ptr)
 └── String*     (_str_val)

StructuredPtr
 └── Array* / Object* / PartialJson* (union)

Document
 ├── _Value (루트 값)
 └── Arena* (메모리 풀)
```

---

## 핵심 클래스

### `_Value`

JSON의 단일 값을 나타내는 핵심 클래스입니다. 정수, 실수, 불리언, 문자열, null, 배열, 객체를 모두 표현합니다.

**헤더**: `claujson.h`

#### 주요 특성

- **복사 불가**: `_Value(const _Value&) = delete`
- **이동 가능**: `_Value(_Value&&) noexcept`
- 내부적으로 union을 통해 타입별 데이터를 공유 저장

#### 내부 저장 구조

```cpp
union {
    int64_t   _int_val;
    uint64_t  _uint_val;
    double    _float_val;
    Array*    _array_ptr;
    Object*   _obj_ptr;
    PartialJson* _pj_ptr;
    String*   _str_val;
    bool      _bool_val;
};
_ValueType _type;
```

#### 생성자

| 생성자 | 설명 |
|--------|------|
| `_Value()` | 기본 생성자 (NONE 타입) |
| `_Value(int x)` | 정수 값 |
| `_Value(unsigned int x)` | 부호 없는 정수 |
| `_Value(int64_t x)` | 64비트 정수 |
| `_Value(uint64_t x)` | 64비트 부호 없는 정수 |
| `_Value(double x)` | 부동소수점 |
| `_Value(bool x)` | 불리언 |
| `_Value(std::nullptr_t x)` | null 값 |
| `_Value(Arena* pool, StringView x)` | 문자열 (UTF-8 검증 포함) |
| `_Value(Arena* pool, const char* x)` | C 문자열 |
| `_Value(Array* x)` | 배열 포인터로부터 |
| `_Value(Object* x)` | 객체 포인터로부터 |

#### 타입 확인 메서드

| 메서드 | 반환 | 설명 |
|--------|------|------|
| `is_valid()` | `bool` | 유효한 값인지 확인 |
| `is_null()` | `bool` | null 타입인지 확인 |
| `is_primitive()` | `bool` | 기본 타입인지 확인 |
| `is_structured()` | `bool` | 배열 또는 객체인지 확인 |
| `is_array()` | `bool` | 배열인지 확인 |
| `is_object()` | `bool` | 객체인지 확인 |
| `is_int()` | `bool` | 64비트 정수인지 확인 |
| `is_uint()` | `bool` | 부호 없는 64비트 정수인지 확인 |
| `is_float()` | `bool` | 부동소수점인지 확인 |
| `is_number()` | `bool` | 숫자 타입인지 확인 (int, uint, float) |
| `is_bool()` | `bool` | 불리언인지 확인 |
| `is_str()` | `bool` | 문자열인지 확인 |
| `is_virtual()` | `bool` | 가상 노드인지 확인 (내부 파싱용) |
| `is_partial_json()` | `bool` | 부분 JSON인지 확인 (내부용) |

#### 값 접근 메서드

| 메서드 | 반환 | 설명 |
|--------|------|------|
| `int_val()` | `int64_t` | 정수 값 반환 |
| `uint_val()` | `uint64_t` | 부호 없는 정수 값 반환 |
| `float_val()` | `double` | 부동소수점 값 반환 |
| `bool_val()` | `bool` | 불리언 값 반환 |
| `str_val()` | `String&` | 문자열 객체 참조 반환 |
| `get_integer()` | `int64_t` | 정수 값 반환 (`int_val()` 별칭) |
| `get_unsigned_integer()` | `uint64_t` | 부호 없는 정수 반환 |
| `get_floating()` | `double` | 부동소수점 반환 |
| `get_boolean()` | `bool` | 불리언 반환 |
| `get_string()` | `String&` | 문자열 반환 |
| `get_number<T>()` | `T` | 숫자를 지정 타입으로 변환 반환 |

#### 구조체 접근 메서드

| 메서드 | 반환 | 설명 |
|--------|------|------|
| `as_array()` | `Array*` | Array 포인터 반환 (배열이 아니면 nullptr) |
| `as_object()` | `Object*` | Object 포인터 반환 (객체가 아니면 nullptr) |
| `as_partial_json()` | `PartialJson*` | PartialJson 포인터 반환 |
| `as_structured()` | `StructuredPtr` | 구조체 포인터 래퍼 반환 |
| `as_structured_ptr()` | `StructuredPtr` | 구조체 포인터 래퍼 반환 |

#### 연산자

| 연산자 | 설명 |
|--------|------|
| `operator[](uint64_t idx)` | 인덱스로 자식 값 접근 |
| `operator[](const _Value& key)` | 키로 자식 값 접근 (객체) |
| `operator bool()` | 유효성 확인 |
| `operator==` / `operator!=` | 비교 |
| `operator<` | 정렬용 비교 |
| `operator=(_Value&&)` | 이동 대입 |

#### 값 설정 메서드

| 메서드 | 설명 |
|--------|------|
| `set_int(long long x)` | 정수 값 설정 |
| `set_uint(unsigned long long x)` | 부호 없는 정수 설정 |
| `set_float(double x)` | 부동소수점 설정 |
| `set_str(Arena*, const char*, uint64_t)` | 문자열 설정 (UTF-8 검증, 이스케이프 처리) |
| `set_bool(bool x)` | 불리언 설정 |
| `set_null()` | null 설정 |
| `set_none()` | NONE 타입으로 초기화 |

#### JSON 포인터 접근

```cpp
_Value& json_pointerB(const my_vector<_Value>& routeVec);
const _Value& json_pointerB(const my_vector<_Value>& routeVec) const;
```

경로(route)를 따라 중첩된 값에 접근합니다. 배열은 인덱스(정수), 객체는 키(문자열)를 사용합니다.

#### 복제

```cpp
_Value clone(Arena* pool) const;
```

값을 깊은 복사(deep copy)합니다. `pool`이 제공되면 Arena에 할당합니다.

---

### `Array`

JSON 배열을 나타내는 클래스입니다.

**헤더**: `claujson_array.h`

#### 팩토리 메서드

| 메서드 | 설명 |
|--------|------|
| `static _Value Make(Arena* pool)` | 일반 배열 생성 |
| `static _Value MakeVirtual(Arena* pool)` | 가상 배열 생성 (내부 파싱용) |

#### 주요 메서드

| 메서드 | 반환 | 설명 |
|--------|------|------|
| `size()` | `uint64_t` | 원소 수 반환 |
| `empty()` | `bool` | 비어있는지 확인 |
| `get_data_size()` | `uint64_t` | 원소 수 반환 |
| `get_value_list(idx)` | `_Value&` | 인덱스로 값 접근 |
| `operator[](idx)` | `_Value&` | 인덱스로 값 접근 |
| `add_element(Value val)` | `bool` | 원소 추가 |
| `assign_element(idx, Value val)` | `bool` | 인덱스 위치에 값 대입 |
| `insert(idx, Value val)` | `bool` | 인덱스 위치에 삽입 |
| `erase(idx, bool real)` | `void` | 인덱스 위치 원소 삭제 |
| `erase(const _Value&, bool real)` | `void` | 값으로 검색 후 삭제 |
| `find(const _Value&, start)` | `uint64_t` | 값 검색, 없으면 `npos` |
| `clear()` | `void` | 전체 초기화 |
| `clear(idx)` | `void` | 인덱스 위치 원소 초기화 |
| `reserve_data_list(len)` | `void` | 용량 예약 |
| `clone(Arena*)` | `_Value` | 깊은 복사 |
| `is_virtual()` | `bool` | 가상 노드 여부 |
| `get_parent()` | `StructuredPtr` | 부모 노드 반환 |
| `set_parent(StructuredPtr)` | `void` | 부모 노드 설정 |

#### 이터레이터

```cpp
_ValueIterator  begin();   // _Value*
_ValueIterator  end();
_ConstValueIterator begin() const;
_ConstValueIterator end()   const;
```

---

### `Object`

JSON 객체(키-값 쌍의 컬렉션)를 나타내는 클래스입니다.

**헤더**: `claujson_object.h`

#### 팩토리 메서드

| 메서드 | 설명 |
|--------|------|
| `static _Value Make(Arena* pool)` | 일반 객체 생성 |
| `static _Value MakeVirtual(Arena* pool)` | 가상 객체 생성 (내부 파싱용) |

#### 주요 메서드

| 메서드 | 반환 | 설명 |
|--------|------|------|
| `size()` | `uint64_t` | 키-값 쌍의 수 반환 |
| `empty()` | `bool` | 비어있는지 확인 |
| `get_data_size()` | `uint64_t` | 키-값 쌍의 수 반환 |
| `get_value_list(idx)` | `_Value&` | 인덱스로 값 접근 |
| `get_key_list(idx)` | `_Value&` | 인덱스로 키 접근 (private) |
| `get_const_key_list(idx)` | `const _Value&` | 인덱스로 키 읽기 |
| `operator[](const _Value& key)` | `_Value&` | 키로 값 접근 |
| `operator[](uint64_t idx)` | `_Value&` | 인덱스로 값 접근 |
| `find(const _Value& key)` | `uint64_t` | 키 검색, 없으면 `npos` |
| `add_element(Value key, Value val)` | `bool` | 키-값 쌍 추가 |
| `assign_value_element(idx, Value val)` | `bool` | 인덱스 위치에 값 대입 |
| `change_key(const _Value&, Value)` | `bool` | 키 변경 |
| `change_key(uint64_t idx, Value)` | `bool` | 인덱스로 키 변경 |
| `erase(const _Value& key, bool real)` | `void` | 키로 삭제 |
| `erase(uint64_t idx, bool real)` | `void` | 인덱스로 삭제 |
| `chk_key_dup(uint64_t*)` | `bool` | 키 중복 확인 |
| `clear()` | `void` | 전체 초기화 |
| `reserve_data_list(len)` | `void` | 용량 예약 |
| `clone(Arena*)` | `_Value` | 깊은 복사 |

#### 이터레이터

```cpp
// Pair<_Value, _Value>*  (키-값 쌍 포인터)
_ValueIterator      begin();
_ValueIterator      end();
_ConstValueIterator begin() const;
_ConstValueIterator end()   const;
```

---

### `PartialJson`

멀티스레드 파싱 과정에서 분할된 JSON 조각을 임시로 보관하는 내부 클래스입니다.

**헤더**: `claujson_partialjson.h`

> **주의**: 이 클래스는 `LoadData` / `LoadData2` 내부에서만 사용되며, 일반 사용자는 직접 다루지 않습니다.

#### 특징

- 배열(`arr_vec`) 또는 객체(`obj_data`) 중 하나로 작동
- `virtualJson`: 가상 루트 노드(분할 경계에서 발생)를 보관
- 부모가 `nullptr`인 최상위 부분 JSON

#### 주요 메서드

| 메서드 | 설명 |
|--------|------|
| `add_array_element(Value val)` | 배열 원소 추가 |
| `add_object_element(Value key, Value val)` | 객체 원소 추가 |
| `get_data_size()` | 보관 중인 원소 수 |
| `get_value_list(idx)` | 인덱스로 값 접근 |
| `get_key_list(idx)` | 인덱스로 키 접근 |
| `MergeWith(Array*, int)` | Array와 병합 |
| `MergeWith(PartialJson*, int)` | PartialJson과 병합 |

---

### `StructuredPtr`

`Array*`, `Object*`, `PartialJson*`를 하나의 타입으로 다루는 포인터 래퍼입니다.

**헤더**: `claujson.h`

#### 생성자

```cpp
StructuredPtr()                    // nullptr
StructuredPtr(Array* arr)          // type = 1
StructuredPtr(Object* obj)         // type = 2
StructuredPtr(PartialJson* pj)     // type = 3
StructuredPtr(_Value& x)           // _Value로부터 자동 판별
StructuredPtr(const _Value& x)
```

#### 타입 확인

```cpp
bool is_array()        const;
bool is_object()       const;
bool is_partial_json() const;
bool is_nullptr()      const;
bool is_user_type()    const;  // array || object
bool is_virtual()      const;
explicit operator bool() const;
```

#### 주요 메서드

| 메서드 | 설명 |
|--------|------|
| `size()` / `empty()` | 크기 확인 |
| `get_data_size()` | 원소 수 |
| `get_value_list(idx)` | 값 접근 |
| `get_key_list(idx)` | 키 접근 (객체/PartialJson) |
| `get_const_key_list(idx)` | 키 읽기 전용 접근 |
| `add_array_element(Value)` | 배열에 원소 추가 |
| `add_object_element(Value, Value)` | 객체에 키-값 추가 |
| `operator[](const _Value&)` | 키로 값 접근 |
| `operator[](uint64_t)` | 인덱스로 값 접근 |
| `find_by_key(const _Value&)` | 키 검색 |
| `find_by_value(const _Value&, start)` | 값 검색 |
| `insert(idx, Value)` | 배열에 삽입 |
| `erase(idx, bool)` | 인덱스로 삭제 |
| `erase(const _Value&, bool)` | 키로 삭제 |
| `assign_value(idx, Value)` | 인덱스 위치에 값 대입 |
| `change_key(const _Value&, Value&&)` | 키 변경 |
| `change_key(uint64_t, Value&&)` | 인덱스로 키 변경 |
| `chk_key_dup(uint64_t*)` | 키 중복 확인 |
| `get_parent()` | 부모 노드 반환 |
| `null_parent()` | 부모 포인터 null로 설정 |
| `MergeWith(StructuredPtr, int)` | 다른 구조체와 병합 |
| `reserve_data_list(sz)` | 용량 예약 |
| `clear()` | 전체 초기화 |
| `clear(idx)` | 특정 인덱스 초기화 |
| `Delete()` | 포인터 delete 및 nullptr 설정 |
| `get_pool()` | Arena 포인터 반환 |
| `has_pool()` | Arena 사용 여부 |

---

### `Document`

파싱 결과를 보유하는 최상위 컨테이너입니다.

**헤더**: `claujson.h`

```cpp
class Document {
    _Value x;    // 루트 JSON 값
    Arena* pool; // 메모리 풀
};
```

#### 생성자

```cpp
Document(uint64_t size = Arena::initialSize);
Document(_Value&& x, uint64_t size = Arena::initialSize);
Document(Document&& d) noexcept;
```

#### 주요 메서드

| 메서드 | 반환 | 설명 |
|--------|------|------|
| `Get()` | `_Value&` | 루트 값 접근 |
| `Get() const` | `const _Value&` | 루트 값 읽기 전용 접근 |
| `GetAllocator()` | `Arena*` | 메모리 풀 접근 |

---

### `String`

SSO(Small String Optimization)를 지원하는 문자열 클래스입니다.

**헤더**: `claujson_string.h`

#### 특징

- **Short String**: 11바이트(`CLAUJSON_STRING_BUF_SIZE`) 이하는 인라인 버퍼에 저장 (`_ValueType::SHORT_STRING`)
- **Long String**: 11바이트 초과 시 힙(또는 Arena)에 할당 (`_ValueType::STRING`)
- 복사 불가 (`operator=(const String&) = delete`)
- 이동 가능

#### 주요 메서드

| 메서드 | 설명 |
|--------|------|
| `data()` | `char*` 포인터 반환 |
| `size()` | 문자열 길이 반환 |
| `is_valid()` | 유효성 확인 |
| `is_str()` | 문자열 타입인지 확인 |
| `clear()` | 메모리 해제 및 초기화 |
| `clone(Arena*)` | 복사본 생성 |
| `get_std_string(bool&)` | `std::string`으로 변환 |
| `get_string_view(bool&)` | `StringView`로 변환 |
| `find(char, start)` | 문자 검색 |
| `substr(start, len)` | 부분 문자열 생성 |
| `operator==` / `operator<` | 비교 |

---

### `Arena` (메모리 풀)

블록 기반 메모리 풀로, 빈번한 `new`/`delete`를 대체합니다.

**헤더**: `claujson_internal.h`

#### 특징

- 두 종류의 블록 관리: 일반 크기(`no=0`)와 대형 크기(`no=1`)
- 정렬(alignment) 지원: `std::align`을 사용하여 올바른 정렬 보장
- `link_from()`: 여러 Arena를 하나로 연결 (병렬 파싱 후 병합 시 사용)

#### 주요 메서드

```cpp
template <class T>
T* allocate(uint64_t size, uint64_t align = alignof(T));

template <class T>
void deallocate(T* ptr, uint64_t len);   // 현재 no-op (최적화 목적 코드만 존재)

template<typename T, typename... Args>
T* create(Args&&... args);               // allocate + placement new

void Clear();    // 블록 재사용 (해제하지 않음)
void Reset();    // 블록 리셋
void link_from(Arena* other);  // 다른 Arena 블록을 이쪽으로 연결
```

#### 상수

```cpp
static const uint64_t initialSize = 1024 * 1024; // 기본 블록 크기 (1MB)
```

---

## 파서 및 직렬화

### `parser`

JSON을 파싱하는 클래스입니다.

**헤더**: `claujson.h`

```cpp
class parser {
    _simdjson::dom::parser_for_claujson test_;
    std::unique_ptr<ThreadPool> pool;
};
```

#### 생성자

```cpp
parser(int thr_num = 0);  // thr_num=0이면 하드웨어 스레드 수에서 2를 뺀 값 사용
```

#### 파싱 메서드

```cpp
// JSON 파일 파싱
std::pair<bool, uint64_t> parse(const std::string& fileName, Document& d, uint64_t thr_num);

// JSON 문자열 파싱
std::pair<bool, uint64_t> parse_str(StringView str, Document& d, uint64_t thr_num);

// C++20: UTF-8 문자열 파싱
std::pair<bool, uint64_t> parse_str(std::u8string_view str, Document& d, uint64_t thr_num);
```

**반환값**: `{성공 여부, 처리된 토큰 수}`

#### 파싱 파이프라인

```
1. simdjson stage1: 구조 토큰 색인 생성
2. is_valid2(): 병렬 문법 검증 및 count_vec 구성
3. LoadData2::parse(): 병렬 JSON 트리 구성
   ├── 청크별 __LoadData() 병렬 실행 (ThreadPool)
   └── Merge(): 청크 결과 순차 병합
4. 최종 _Value 반환
```

---

### `writer`

JSON을 직렬화하는 클래스입니다.

**헤더**: `claujson.h`

```cpp
class writer {
    std::unique_ptr<ThreadPool> pool;
};
```

#### 생성자

```cpp
writer(int thr_num = 0);
```

#### 직렬화 메서드

| 메서드 | 설명 |
|--------|------|
| `write_to_str(const _Value&, bool pretty)` | JSON 문자열로 직렬화 |
| `write_to_str2(const _Value&, bool pretty)` | JsonView 방식으로 직렬화 (실험적) |
| `write(const string&, const _Value&, bool pretty)` | 파일로 직렬화 |
| `write_parallel(Arena*, string&, _Value&, uint64_t, bool)` | 트리 분할 병렬 직렬화 |
| `write_parallel2(string&, const _Value&, uint64_t, bool)` | JsonView 기반 병렬 직렬화 |
| `write_parallel3(Arena*, string&, _Value&, uint64_t, bool)` | NodeView 기반 병렬 직렬화 (실험적-테스트중..) |

#### 직렬화 방식 비교

| 방식 | 클래스 | 멀티스레드 | 특징 |
|------|--------|-----------|------|
| `write_to_str` | `LoadData2::_write` | No | 재귀적 DFS 순회 |
| `write_to_str2` | `JsonView` + `print` | 가능 | 배열 기반 선형 순회 |
| `write_parallel` | `LoadData2::write_` | Yes | 트리 분할 후 청크별 직렬화 |
| `write_parallel2` | `JsonView` + `print` | Yes | 선형 JsonView 배열 분할 |

---

## 유틸리티

### `diff` / `patch`

두 JSON 값의 차이를 계산하고 적용합니다.

```cpp
[[nodiscard]]
_Value diff(Arena* pool, const _Value& x, const _Value& y);

_Value& patch(Arena* pool, _Value& x, const _Value& diff);
```

#### diff 출력 형식

diff 결과는 JSON Patch(RFC 6902 유사) 형식의 배열입니다:

```json
[
  { "op": "replace", "path": [...], "value": <new_value> },
  { "op": "remove",  "path": [...], "last_key": "key" },
  { "op": "remove",  "path": [...], "last_idx": 3 },
  { "op": "add",     "path": [...], "key": "key", "value": <value> },
  { "op": "add",     "path": [...], "value": <value> }
]
```

---

### 문자열 변환 함수

```cpp
/*
// JSON 문자열 내 이스케이프 처리 (예: \n -> \\n)
std::pair<bool, std::string> convert_to_string_in_json(StringView x);

// JSON 문자열로서 유효한지 확인
bool is_valid_string_in_json(StringView x);

// 숫자 문자열을 _Value로 변환
bool convert_number(StringView x, claujson::_Value& data);

// 문자열을 _Value로 변환
bool convert_string(StringView x, claujson::_Value& data);
*/
```

#### C++20 (char8_t) 지원

```cpp
/*
#if __cpp_lib_char8_t
std::pair<bool, std::string> convert_to_string_in_json(std::u8string_view x);
bool is_valid_string_in_json(std::u8string_view x);
#endif
*/
```

---

### `clean`

```cpp
void clean(_Value& x);
```

`_Value`의 메모리를 해제합니다. Arena를 사용하는 경우 소멸자만 호출하고, 힙 할당의 경우 `delete`합니다.

> **참고**: 현재 구현에서는 즉시 반환(`return`)하여 실제 해제를 수행하지 않습니다. Arena의 생명주기에 따라 일괄 해제를 기대하는 설계입니다.

---

### `Log` 시스템

```cpp
claujson::Log log;          // 전역 로그 객체
claujson::Log::Info info;   // 정보 레벨 태그
claujson::Log::Warning warn; // 경고 레벨 태그
```

#### 사용 예시

```cpp
claujson::log.console();        // 콘솔 출력 활성화
claujson::log.info();           // INFO 레벨 활성화
claujson::log.warn();           // WARN 레벨 활성화

claujson::log << claujson::info << "파싱 시작\n";
claujson::log << claujson::warn << "오류 발생\n";
```

#### 출력 옵션

| 옵션 | 설명 |
|------|------|
| `log.console()` | 콘솔 출력 |
| `log.file()` | 파일 출력 |
| `log.console_and_file()` | 콘솔 + 파일 |
| `log.no_print()` | 출력 비활성화 |
| `log.file_name(str)` | 로그 파일명 설정 |

---

## 타입 시스템

### `_ValueType` 열거형

```cpp
enum class _ValueType : int32_t {
    NONE = 0,        // 초기화되지 않은 상태
    ARRAY,           // JSON 배열
    OBJECT,          // JSON 객체
    PARTIAL_JSON,    // 부분 JSON (내부용)
    INT,             // 64비트 부호 있는 정수
    UINT,            // 64비트 부호 없는 정수
    FLOAT,           // IEEE 754 double
    BOOL,            // 불리언
    NULL_,           // JSON null
    STRING,          // 문자열 (힙/Arena 할당)
    SHORT_STRING,    // 문자열 (11바이트 이하 인라인 저장)
    NOT_VALID,       // 유효하지 않은 값 (접근 불가 마커)
    ERROR            // 오류 상태
};
```

### `Value` 래퍼 클래스

`_Value`를 안전하게 전달하기 위한 래퍼입니다.

```cpp
class Value {
    _Value x;
public:
    Value(_Value&& x) noexcept;
    Value(Value&& x) noexcept;
    _Value& Get() noexcept;
    const _Value& Get() const noexcept;
};
```

> `add_element`, `add_object_element` 등의 메서드는 `Value`를 인자로 받아 소유권을 이전합니다.

---

## 메모리 관리

### Arena 기반 할당

```cpp
Arena pool;

// Array 생성 - pool 할당
_Value arr = Array::Make(&pool);

// Object 생성 - pool 할당
_Value obj = Object::Make(&pool);

// 문자열 생성 - pool 할당
_Value str(&pool, "hello"sv);
```

### 할당 전략

```
크기 < defaultBlockSize(1MB) → Block[0] (일반 블록)
크기 >= defaultBlockSize    → Block[1] (대형 블록)
```

### 병렬 파싱에서의 메모리 병합

```
메인 pool
  └── link_from(sub_pool_1)
  └── link_from(sub_pool_2)
  └── link_from(sub_pool_3)
```

각 스레드는 독립적인 `Arena`를 사용하여 파싱하고, 완료 후 메인 `Arena`에 블록을 연결합니다. 이를 통해 스레드 간 메모리 경합을 제거합니다.

---

## 사용 예시

### 기본 파싱

```cpp
#include "claujson.h"

// 로그 설정 (선택)
claujson::log.console();
claujson::log.warn();

// 파서 생성
claujson::parser p;
claujson::Document doc;

// 파일 파싱
auto [ok, token_count] = p.parse("data.json", doc, 4);
if (!ok) {
    // 오류 처리
    return;
}

claujson::_Value& root = doc.Get();
```

### 문자열 파싱

```cpp
claujson::parser p;
claujson::Document doc;

auto [ok, _] = p.parse_str(R"({"name": "Alice", "age": 30})", doc, 1);
if (ok) {
    auto& root = doc.Get();
    // root는 Object
}
```

### 값 접근

```cpp
claujson::_Value& root = doc.Get();

// 객체 접근
if (root.is_object()) {
    claujson::Arena arena;
    claujson::_Value key(&arena, "name");
    
    claujson::_Value& name = root[key];
    if (name.is_str()) {
        bool fail = false;
        std::string s = name.str_val().get_std_string(fail);
    }
}

// 배열 접근
if (root.is_array()) {
    uint64_t sz = root.as_array()->size();
    for (uint64_t i = 0; i < sz; ++i) {
        claujson::_Value& elem = root[i];
        if (elem.is_int()) {
            int64_t v = elem.int_val();
        }
    }
}
```

### 값 생성 및 조작

```cpp
claujson::Arena pool;

// 배열 생성
claujson::_Value arr = claujson::Array::Make(&pool);
arr.as_array()->add_element(claujson::Value(claujson::_Value(42LL)));
arr.as_array()->add_element(claujson::Value(claujson::_Value(true)));
arr.as_array()->add_element(claujson::Value(claujson::_Value(&pool, "hello"sv)));

// 객체 생성
claujson::_Value obj = claujson::Object::Make(&pool);
obj.as_object()->add_element(
    claujson::Value(claujson::_Value(&pool, "key"sv)),
    claujson::Value(claujson::_Value(123LL))
);
```

### JSON 직렬화

```cpp
claujson::writer w;

// 문자열로 변환
std::string json_str = w.write_to_str(root, false);       // compact
std::string json_pretty = w.write_to_str(root, true);     // pretty

// 파일로 저장
w.write("output.json", root, false);

// 병렬 직렬화 (대용량)
Arena* pool = doc.GetAllocator();
w.write_parallel(pool, "output.json", root, 4, false);
```

### diff / patch

```cpp
claujson::Arena pool;
claujson::_Value d = claujson::diff(&pool, original, modified);

// diff 적용
claujson::_Value& patched = claujson::patch(&pool, original, d);
```

---

## 내부 구현 상세

### `Pointer` 클래스

포인터에 2비트의 메타데이터를 인코딩하는 클래스입니다 (포인터 태깅 기법).

```cpp
class Pointer {
    void* ptr;
    // bit 63     : left_type  (is_virtual 여부)
    // bits 1-0   : right_type (1=Array, 2=Object, 3=PartialJson)
};
```

`Array`와 `Object`의 `parent` 필드에 사용됩니다.

### `my_vector<T>` (`Vector2<T>`)

Arena를 지원하는 커스텀 동적 배열입니다.

```cpp
template <class T>
using my_vector = Vector2<T>;
```

주요 기능:
- Arena 할당/해제 지원
- `Divide(start_idx)`: 지정 인덱스부터 끝까지 분리하여 새 벡터 반환
- `insert(start, end)`: 이동 시맨틱으로 원소 삽입

### `LoadData2` 내부 클래스

병렬 파싱 및 직렬화의 핵심 로직을 담당합니다.

**주요 내부 메서드**:

| 메서드 | 설명 |
|--------|------|
| `__LoadData(...)` | 정적 함수. 단일 청크를 파싱하여 PartialJson 구성 |
| `Merge(next, ut, ut_next)` | 두 PartialJson을 병합 |
| `Merge2(next, ut, ut_next, op)` | `Merge`의 변형 (null ut 처리 추가) |
| `Divide(pool, pos, result)` | 트리의 특정 위치에서 분할하여 PartialJson 생성 |
| `Divide2(pool, n, j, result, hint)` | n개의 분할점 계산 및 분할 실행 |
| `Find2(root, n, idx, ...)` | 균등 분할점 탐색 (재귀) |
| `Size(Array*)` | 서브트리의 노드 수 계산 |
| `Size2(const _Value&)` | JsonView용 노드 수 계산 |
| `_write(...)` | 재귀 DFS 직렬화 |
| `_write2(...)` | 반복 스택 기반 직렬화 |

### `is_valid2` 함수

simdjson의 구조 토큰 배열을 기반으로 JSON 문법 유효성을 검사합니다. goto 기반 상태 머신으로 구현되어 있습니다.

**상태 목록**:
- `object_begin` / `object_field` / `object_continue`
- `array_begin` / `array_value` / `array_continue`
- `scope_end` / `document_end`

동시에 각 배열/객체의 원소 수를 `count_vec`에 기록하여 이후 `reserve_data_list()` 최적화에 활용합니다.

### `JsonView` 배열

`write_parallel2` / `write_to_str2`에서 사용하는 선형 뷰 표현입니다.

```cpp
struct JsonView {
    Pointer value;
    // right_type: 0=ARRAY_START, 1=OBJECT_START, 2=KEY, 3=VALUE
    // left_type:  1 = END(ARRAY/OBJECT)
};
```

JSON 트리를 DFS 순서로 펼쳐 배열에 저장한 뒤, 원하는 구간을 여러 스레드에 분배하여 병렬 직렬화를 수행합니다.

---

*본 문서는 claujson 소스 코드 분석을 기반으로 작성되었습니다.*
