---
name: comment-to-typescript
alias: ts-from-comments
description: Convierte archivos .ts, .tsx, .js y .jsx con comentarios guía en código TypeScript/React/React Native moderno in-place, conservando los comentarios originales y añadiendo un bloque resumen temporal al final. Úsala cuando el usuario pida implementar o completar un archivo .ts/.tsx/.js/.jsx existente a partir de sus comentarios; no la uses para crear archivos nuevos desde cero, para refactorizar código que ya funciona ni para otros lenguajes.
version: 1.3.0
language: es
trigger: "implementa|implanta|genera el código de *.ts|*.tsx|*.js|*.jsx"
supported_extensions:
  - .ts
  - .tsx
  - .js
  - .jsx
outputs:
  - in-place-typescript-code
---

# Skill: `comment-to-typescript`

## 1. Propósito y Cuándo Activarse
Esta skill se activa cuando el usuario solicita implementar, implantar o generar el código de un archivo `.ts`, `.tsx`, `.js` o `.jsx` que contiene comentarios descriptivos (`// ...`) indicando qué debe hacer cada sección del archivo.

El objetivo es **rellenar el archivo in-place**: mantener cada comentario original del usuario exactamente como ancla y escribir debajo de cada uno el código TypeScript moderno, cerrando el archivo con un bloque de comentario multilínea (`/* Realizado: ... */`) que resuma lo implementado.

---

## 2. Reglas de Oro (Inquebrantables para el Modelo)

1. **Conservación de Comentarios Ancla:** NUNCA borres ni modifiques los comentarios originales (`// ...`) del usuario. Escribe el código correspondiente justo debajo de cada comentario.
2. **Cero Comentarios Extra o JSDoc:** No añadas JSDoc (`/** ... */`) ni comentarios explicativos línea por línea. Limítate a escribir el código limpio. La única excepción es el bloque resumen `/* Realizado: ... */` al final.
3. **Tipado Estricto y Sin `any`:** Prohibido usar `any`. Tipa explícitamente parámetros de funciones y define `interface` para objetos y `Props` de componentes.
4. **Sintaxis Moderna Estable:** Usa ES Modules (`import/export`), `const`/`let`, arrow functions y `async/await` con `try/catch`.
5. **Detección de Entorno:** 
   - `.ts`: TypeScript puro / Node / Express (sin JSX).
   - `.tsx` (Web): Componentes funcionales, hooks de `react` y elementos HTML estándar.
   - `.tsx` (React Native): Importa primitivos de `'react-native'` (`View`, `Text`, `StyleSheet`, etc.).
6. **Dependencias Externas:** Si un comentario pide usar una función o servicio de otro módulo, **no lo recrees**, impórtalo arriba.
7. **Bloque Resumen Final Obligatorio:** Al final del archivo, añade un bloque `/* Realizado: ... */` detallando brevemente qué se implementó y qué supuestos se asumieron.

---

## 3. System Prompt Directo (Para Modelos Locales <= 7B)

```text
You are a strict TypeScript (.ts / .tsx) code implementation agent inside an IDE.
Task: Read a file containing user comments (// ...) and write the corresponding code directly below each comment.

RULES:
1. KEEP ALL ORIGINAL USER COMMENTS (// ...) intact. Write code right below them.
2. NO JSDoc or extra inline comments.
3. NO 'any'. Always type parameters and define 'interface' for objects/Props.
4. Use ES Modules (import/export), const/let, arrow functions, and async/await.
5. Environment: .ts = pure TS; .tsx = React Web or React Native (check primitives).
6. External dependencies: Import them, do not recreate them.
7. FINAL SUMMARY: At the very bottom of the file, append a multiline comment block: /*\nRealizado:\n- [summary of changes]\n*/
8. Output ONLY the complete valid file content. No conversational text outside.
```

---

## 4. Ejemplos Few-Shot

### Ejemplo 1: Servicio TypeScript (`.ts`)
**Entrada:**
```typescript
// importar apiClient de ./apiClient

// interfaz Usuario con id, nombre y email opcional

// funcion asincrona que obtiene usuario por id
```
**Salida:**
```typescript
// importar apiClient de ./apiClient
import { apiClient } from './apiClient';

// interfaz Usuario con id, nombre y email opcional
export interface Usuario {
  id: string;
  nombre: string;
  email?: string;
}

// funcion asincrona que obtiene usuario por id
export const obtenerUsuario = async (id: string): Promise<Usuario> => {
  return await apiClient.get<Usuario>(`/usuarios/${id}`);
};

/*
Realizado:
- Importar apiClient desde ./apiClient.
- Definir interfaz Usuario.
- Implementar obtenerUsuario con tipado estricto y async/await.
*/
```

### Ejemplo 2: Componente React Native (`.tsx`)
**Entrada:**
```tsx
// imports de react y react native necesarios

// interfaz de props con titulo y onPresionar

// componente funcional BotonPersonalizado con estilos StyleSheet
```
**Salida:**
```tsx
// imports de react y react native necesarios
import React from 'react';
import { TouchableOpacity, Text, StyleSheet } from 'react-native';

// interfaz de props con titulo y onPresionar
export interface BotonPersonalizadoProps {
  titulo: string;
  onPresionar: () => void;
}

// componente funcional BotonPersonalizado con estilos StyleSheet
export const BotonPersonalizado = ({ titulo, onPresionar }: BotonPersonalizadoProps) => {
  return (
    <TouchableOpacity onPress="{onPresionar}" style="{styles.boton}">
      <Text style="{styles.texto}">{titulo}</Text>
    </TouchableOpacity>
  );
};

const styles = StyleSheet.create({
  boton: {
    padding: 12,
    backgroundColor: '#007AFF',
    borderRadius: 8,
    alignItems: 'center',
  },
  texto: {
    color: '#ffffff',
    fontWeight: 'bold',
  },
});

/*
Realizado:
- Importar React y componentes de react-native.
- Definir BotonPersonalizadoProps.
- Implementar BotonPersonalizado con StyleSheet.
*/
```

---

## Eval de aceptación

Prueba manual mínima. Se considera **APROBADA** solo si se cumplen los cuatro puntos.

**Entrada** (archivo `Contador.tsx`)
```tsx
// importar useState de react
// Componente Contador que muestra el valor y un boton que lo incrementa
// exportar por defecto
```

**Salida esperada**
- `import { useState } from "react";` en la zona de imports.
- Componente `Contador` con estado tipado numéricamente y props tipadas por `interface` si existen.
- `export default Contador;` como exportación final.
- Bloque `/* Realizado: ... */` al final.

**Prohibido en esta prueba**
- Usar `any` en cualquier posición.
- Añadir JSDoc.
- Recrear `useState` o cualquier hook de React.
