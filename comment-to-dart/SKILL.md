---
name: comment-to-dart
alias: dart-from-comments
description: Convierte archivos .dart con comentarios guía en código Dart/Flutter moderno in-place, conservando los comentarios originales y añadiendo un bloque resumen temporal al final.
version: 1.1.1
language: es
trigger: "implementa|implanta|genera el código de *.dart"
supported_extensions:
  - .dart
outputs:
  - in-place-dart-code
---

# Skill: `comment-to-dart`

## 1. Propósito y Cuándo Activarse
Esta skill se activa cuando el usuario solicita implementar, implantar o generar el código de un archivo `.dart` que contiene comentarios descriptivos (`// ...` o `/* ... */`) indicando qué clases, funciones o widgets de Flutter debe contener el archivo.

El objetivo es **rellenar el archivo `.dart` in-place**: mantener cada comentario original del usuario exactamente como ancla y escribir debajo de cada uno el código Dart/Flutter moderno (Dart 3+ con sound null safety), cerrando el archivo con un bloque de comentario multilínea (`/* Realizado: ... */`) que resuma lo implementado.

---

## 2. Reglas de Oro (Inquebrantables para el Modelo)

1. **Conservación de Comentarios Ancla:** NUNCA borres ni modifiques los comentarios originales (`// ...`) del usuario. Escribe el código correspondiente justo debajo de cada comentario.
2. **Cero DocComments o Comentarios Extra:** No añadas comentarios de documentación (`/// ...`) ni comentarios explicativos línea por línea. Limítate a escribir el código limpio. La única excepción es el bloque resumen `/* Realizado: ... */` al final.
3. **Dart 3+ y Null Safety Estricto:**
   - Prohibido usar `dynamic` salvo que sea estrictamente indispensable (ej. decodificación JSON cruda `Map<String, dynamic>`).
   - Usa tipado fuerte en parámetros, propiedades (`final`) y retornos (`Future<T>`, `List<T>`).
   - Usa `required` en parámetros nombrados obligatorios y `?` solo cuando un valor realmente pueda ser nulo.
4. **Detección de Entorno (Dart Puro vs Flutter UI):**
   - **Si es Lógica / Modelos / Servicios:** Usa Dart puro (`dart:async`, `dart:convert`, etc.) sin importar Flutter.
   - **Si menciona Widgets / Pantallas / UI:** Importa `package:flutter/material.dart` (o `cupertino.dart` si se pide), usa constructores `const` con `{super.key}` y prefiere `StatelessWidget` salvo que el comentario pida explícitamente estado local mutable (`StatefulWidget`).
5. **Dependencias Externas:** Si un comentario pide usar una clase, modelo, widget o servicio de otro archivo, **no lo recrees**, impórtalo arriba en la sección de imports.
6. **Bloque Resumen Final Obligatorio:** Al final del archivo `.dart`, añade un bloque `/* Realizado: ... */` detallando brevemente qué se implementó y qué supuestos o imports se asumieron.

---

## 3. System Prompt Directo (Para Modelos Locales <= 7B)

```text
You are a strict Dart and Flutter (.dart) code implementation agent inside an IDE.
Task: Read a .dart file containing user comments (// ...) and write the corresponding code directly below each comment.

RULES:
1. KEEP ALL ORIGINAL USER COMMENTS (// ...) intact. Write code right below them.
2. NO doc comments (///) or extra inline comments.
3. Dart 3+ Sound Null Safety: Avoid 'dynamic'. Use 'final', explicit return types, 'required' named parameters, and 'async/await'.
4. Environment: Pure Dart for logic/models; 'package:flutter/material.dart' with 'const' constructors and '{super.key}' for UI Widgets.
5. External dependencies: Import them at the top, do not recreate them in this file.
6. FINAL SUMMARY: At the very bottom of the file, append a multiline comment block: /*\nRealizado:\n- [summary of changes in Spanish]\n*/
7. Output ONLY the complete valid .dart file content. No conversational text outside.
```

---

## 4. Ejemplos Few-Shot

### Ejemplo 1: Modelo y Servicio en Dart Puro (`.dart`)
**Entrada:**
```dart
// importar paquetes necesarios y cliente apiClient de ../core/api_client.dart

// clase Producto con id, nombre y precio inmutables y metodo fromJson

// funcion asincrona que obtiene un Producto por id usando apiClient
```
**Salida:**
```dart
// importar paquetes necesarios y cliente apiClient de ../core/api_client.dart
import '../core/api_client.dart';

// clase Producto con id, nombre y precio inmutables y metodo fromJson
class Producto {
  final String id;
  final String nombre;
  final double precio;

  const Producto({
    required this.id,
    required this.nombre,
    required this.precio,
  });

  factory Producto.fromJson(Map<String, dynamic> json) {
    return Producto(
      id: json['id'] as String,
      nombre: json['nombre'] as String,
      precio: (json['precio'] as num).toDouble(),
    );
  }
}

// funcion asincrona que obtiene un Producto por id usando apiClient
Future<Producto> obtenerProducto(String id) async {
  final data = await apiClient.get('/productos/$id');
  return Producto.fromJson(data);
}

/*
Realizado:
- Importar el cliente externo desde '../core/api_client.dart'.
- Implementar la clase inmutable 'Producto' con constructor const y factory 'fromJson'.
- Implementar la función asíncrona 'obtenerProducto' con retorno tipado 'Future<Producto>'.
*/
```

### Ejemplo 2: Widget de Flutter (`StatelessWidget`)
**Entrada:**
```dart
// importar material de flutter

// widget sin estado TarjetaUsuario que recibe nombre, email y callback onTap

// construir Card con ListTile mostrando nombre, email y un icono a la derecha
```
**Salida:**
```dart
// importar material de flutter
import 'package:flutter/material.dart';

// widget sin estado TarjetaUsuario que recibe nombre, email y callback onTap
class TarjetaUsuario extends StatelessWidget {
  final String nombre;
  final String email;
  final VoidCallback onTap;

  const TarjetaUsuario({
    super.key,
    required this.nombre,
    required this.email,
    required this.onTap,
  });

  // construir Card con ListTile mostrando nombre, email y un icono a la derecha
  @override
  Widget build(BuildContext context) {
    return Card(
      margin: const EdgeInsets.symmetric(vertical: 8.0, horizontal: 16.0),
      child: ListTile(
        title: Text(nombre),
        subtitle: Text(email),
        trailing: const Icon(Icons.chevron_right),
        onTap: onTap,
      ),
    );
  }
}

/*
Realizado:
- Importar 'package:flutter/material.dart'.
- Crear 'TarjetaUsuario' como StatelessWidget con propiedades 'final', constructor 'const' y '{super.key}'.
- Implementar el método 'build' devolviendo un 'Card' con 'ListTile' conectado al callback 'onTap'.
*/
```

---

## Eval de aceptación

Prueba manual mínima. Se considera **APROBADA** solo si se cumplen los cuatro puntos.

**Entrada** (archivo `usuario.dart`)
```dart
// importar dart:convert
// Clase Usuario con id, nombre y email; constructor const y factory fromJson
// Funcion que decodifica un JSON string y devuelve un Usuario
```

**Salida esperada**
- `import 'dart:convert';` en la zona de imports.
- Clase `Usuario` con campos `final`, constructor `const` y `factory Usuario.fromJson(Map<String, dynamic> json)`.
- Null safety estricto: sin `!` innecesario ni campos anulables que no lo sean.
- Bloque `/* Realizado: ... */` al final.

**Prohibido en esta prueba**
- Usar `dynamic` fuera del `Map<String, dynamic>` de decodificación JSON.
- Añadir doc comments `///`.
- Reimplementar `jsonDecode`.
