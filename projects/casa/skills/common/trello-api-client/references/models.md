# Modelos de Datos - Trello API Client

Tipos TypeScript de las entidades del backend (tablas: `users`, `auth_sessions`, `boards`, `lists`, `cards`, `custom_fields`, `card_custom_field_values`).

## Board

```typescript
{
  id: string;
  ownerId: string;
  title: string;
  description: string;
  background: string;      // hex color
  createdAt: string;
  updatedAt: string;
}
```

## BoardList

```typescript
{
  id: string;
  boardId: string;
  title: string;
  position: number;
  createdAt: string;
  updatedAt: string;
}
```

## Card

```typescript
{
  id: string;
  listId: string;
  boardId: string;
  title: string;
  description: string;
  position: number;
  labels: Label[];
  dueDate: string | null;
  createdAt: string;
  updatedAt: string;
}
```

## Label

```typescript
{
  id: string;
  name: string;
  color: string;
}
```

## CustomField

```typescript
{
  id: string;
  boardId: string;
  name: string;
  type: 'text' | 'number' | 'checkbox' | 'date' | 'select';
  options: string[] | null;
  position: number;
  showOnCard: boolean;
  createdAt: string;
  updatedAt: string;
}
```
