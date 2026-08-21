# COM Research Roadmap

## 🎯 Main Topics

### 1. **IUnknown и Reference Counting — утечки и уязвимости**
   - Как неправильный counting приводит к проблемам
   - AddRef/Release механика
   - Double-free и Use-After-Free через COM
   - Memory leak patterns в реальных компонентах
   - **Status**: 🔴 Not started

### 2. **COM object deserialization эксплуатация**
   - ActiveX exploits, как они работают
   - Unmarshaling vulnerabilities
   - Type confusion attacks
   - **Status**: 🔴 Not started

### 3. **COM Type Libraries анализ**
   - Что можно вытащить из .tlb
   - Парсинг programmatically
   - Восстановление сигнатур интерфейсов
   - **Status**: 🔴 Not started

### 4. **Oleviewer/oleaut32 internals**
   - Как OLE Automation работает под капотом
   - IDispatch механика
   - Variant type system
   - **Status**: 🔴 Not started

### 5. **COM Interception через IMessageFilter**
   - Перехват вызовов между потоками
   - Message filtering для анализа
   - Timing attacks
   - **Status**: 🔴 Not started

### 6. **COM Object Substitution**
   - Замена реальных объектов на поддельные для анализа
   - Mock objects для security testing
   - Detection evasion
   - **Status**: 🔴 Not started

---

## 📝 Article Writing Order
1. → IUnknown и Reference Counting
2. → COM Type Libraries анализ
3. → COM object deserialization эксплуатация
4. → Oleviewer/oleaut32 internals
5. → COM Interception через IMessageFilter
6. → COM Object Substitution

---

**Last Updated**: 2026-08-21
