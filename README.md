# Tanstack-Table

```bash
// ============================================
// 1. ИМПОРТЫ - подключаем нужные библиотеки
// ============================================
import React, { useRef, useMemo } from 'react';
import { useReactTable, getCoreRowModel, flexRender } from '@tanstack/react-table';
import { useVirtualizer } from '@tanstack/react-virtual';

// ============================================
// 2. КОМПОНЕНТ - наша таблица
// ============================================
export default function VirtualTable({ data = [] }) {
  // ==========================================
  // 2.1. useRef - создаем "ссылку" на контейнер
  // ==========================================
  // Представьте, что это лазерная указка.
  // Мы говорим React: "запомни этот div, я буду к нему обращаться".
  // Нам это нужно, чтобы виртуализатор знал, где находится скролл.
  const tableContainerRef = useRef(null);

  // ==========================================
  // 2.2. Определяем КОЛОНКИ (схему таблицы)
  // ==========================================
  // useMemo - запоминаем колонки, чтобы они не пересоздавались при каждом рендере.
  // Пустой массив [] означает "создать один раз и больше не менять".
  const columns = useMemo(
    () => [
      {
        // accessorKey - по какому ключу из объекта брать данные
        accessorKey: 'id',
        header: 'ID', // Заголовок в шапке
        size: 50, // Ширина колонки (опционально)
      },
      {
        accessorKey: 'name',
        header: 'Имя',
        size: 200,
      },
      {
        accessorKey: 'age',
        header: 'Возраст',
        size: 100,
      },
      {
        accessorKey: 'city',
        header: 'Город',
        size: 200,
      },
    ],
    [] // Пустой массив = колонки создаются один раз
  );

  // ==========================================
  // 2.3. СОЗДАЕМ ТАБЛИЦУ (TanStack Table)
  // ==========================================
  // Это "мозг" таблицы. Он обрабатывает данные, но НЕ рисует их.
  const table = useReactTable({
    data, // Данные, которые мы получили в пропсах
    columns, // Наша схема
    getCoreRowModel: getCoreRowModel(), // Базовый движок (обязательно!)
  });

  // ==========================================
  // 2.4. СОЗДАЕМ ВИРТУАЛИЗАТОР (TanStack Virtual)
  // ==========================================
  // Это "мозг" виртуализации. Он вычисляет, какие строки видны.
  const virtualizer = useVirtualizer({
    // Сколько всего строк
    count: table.getRowModel().rows.length,
    
    // Где находится скролл? Берем наш div через useRef
    getScrollElement: () => tableContainerRef.current,
    
    // Примерная высота ОДНОЙ строки в пикселях
    // Нужно, чтобы виртуализатор мог рассчитать, сколько строк помещается
    estimateSize: () => 40,
    
    // Сколько "запасных" строк рисовать сверху и снизу
    // Чтобы при быстром скролле не было белого экрана
    overscan: 5,
  });

  // ==========================================
  // 2.5. ПОЛУЧАЕМ ВИРТУАЛЬНЫЕ СТРОКИ
  // ==========================================
  // virtualizer.getVirtualItems() - возвращает ТОЛЬКО видимые строки
  // Например, из 1000 строк он вернет только 15, которые помещаются в экран
  const virtualRows = virtualizer.getVirtualItems();

  // ==========================================
  // 2.6. ВЫСОТА ВСЕХ СТРОК (для скролла)
  // ==========================================
  // Общая высота ВСЕХ строк (например, 1000 * 40px = 40000px)
  // Нужна, чтобы скролл-бар был правильного размера
  const totalSize = virtualizer.getTotalSize();

  // ==========================================
  // 2.7. РЕНДЕРИНГ (РИСУЕМ ТАБЛИЦУ)
  // ==========================================
  return (
    <div style={{ padding: '20px', fontFamily: 'Arial' }}>
      <h2>Таблица с виртуализацией</h2>
      
      {/* 
        ========================================
        КОНТЕЙНЕР со скроллом
        ========================================
        ref={tableContainerRef} - привязываем нашу "лазерную указку"
        style - задаем фиксированную высоту и скролл
      */}
      <div
        ref={tableContainerRef}
        style={{
          height: '500px', // Фиксированная высота, чтобы появился скролл
          overflow: 'auto', // Включаем скролл
          border: '1px solid #ddd',
          borderRadius: '8px',
          position: 'relative', // Важно для позиционирования строк
        }}
      >
        <table
          style={{
            width: '100%',
            borderCollapse: 'collapse',
            position: 'relative', // Чтобы строки позиционировались внутри
          }}
        >
          {/* ===== ШАПКА (THEAD) - всегда видна ===== */}
          <thead
            style={{
              position: 'sticky', // Шапка прилипает при скролле
              top: 0,
              zIndex: 10,
              backgroundColor: '#f8f9fa',
            }}
          >
            {table.getHeaderGroups().map((headerGroup) => (
              <tr key={headerGroup.id}>
                {headerGroup.headers.map((header) => (
                  <th
                    key={header.id}
                    style={{
                      padding: '12px',
                      textAlign: 'left',
                      borderBottom: '2px solid #ddd',
                      fontWeight: 'bold',
                      width: header.column.columnDef.size || 'auto',
                    }}
                  >
                    {flexRender(header.column.columnDef.header, header.getContext())}
                  </th>
                ))}
              </tr>
            ))}
          </thead>

          {/* ===== ТЕЛО (TBODY) - только видимые строки ===== */}
          <tbody>
            {/*
              ============================================
              ВАЖНО: Рисуем ТОЛЬКО virtualRows
              ============================================
            */}
            {virtualRows.map((virtualRow) => {
              // Получаем ОРИГИНАЛЬНУЮ строку из таблицы по индексу
              const realRow = table.getRowModel().rows[virtualRow.index];

              return (
                <tr
                  key={realRow.id}
                  style={{
                    // ===== КЛЮЧЕВОЙ МОМЕНТ №1 =====
                    // Абсолютное позиционирование - строки не влияют друг на друга
                    position: 'absolute',
                    
                    // ===== КЛЮЧЕВОЙ МОМЕНТ №2 =====
                    // Сдвигаем строку вниз на нужное расстояние
                    // Например, 5-я строка сдвигается на 5 * 40px = 200px
                    transform: `translateY(${virtualRow.start}px)`,
                    
                    // ===== КЛЮЧЕВОЙ МОМЕНТ №3 =====
                    // Фиксируем высоту строки
                    height: `${virtualRow.size}px`,
                    
                    // Растягиваем на всю ширину таблицы
                    width: '100%',
                    
                    // Чередование цветов для читаемости
                    backgroundColor: virtualRow.index % 2 === 0 ? '#ffffff' : '#f8f9fa',
                  }}
                >
                  {/* Рисуем ячейки */}
                  {realRow.getVisibleCells().map((cell) => (
                    <td
                      key={cell.id}
                      style={{
                        padding: '8px 12px',
                        borderBottom: '1px solid #eee',
                        whiteSpace: 'nowrap',
                        overflow: 'hidden',
                        textOverflow: 'ellipsis',
                      }}
                    >
                      {flexRender(cell.column.columnDef.cell, cell.getContext())}
                    </td>
                  ))}
                </tr>
              );
            })}
          </tbody>
        </table>

        {/*
          ============================================
          ВАЖНО: ПУСТОЙ БЛОК для создания скролла
          ============================================
          Этот div НЕВИДИМ, но он занимает место.
          totalSize = количество строк * высота строки (например, 40000px)
          Именно он заставляет скролл-бар быть большим.
        */}
        <div
          style={{
            height: totalSize, // Общая высота всех строк
            width: '1px', // Минимальная ширина, чтобы не влиять на макет
            pointerEvents: 'none', // Чтобы не перехватывать клики
          }}
        />
      </div>

      {/* ===== ИНФОРМАЦИЯ ДЛЯ НАГЛЯДНОСТИ ===== */}
      <div style={{ marginTop: '16px', padding: '12px', background: '#f1f3f5', borderRadius: '8px' }}>
        <p style={{ margin: '4px 0' }}>
          <strong>Всего строк:</strong> {table.getRowModel().rows.length}
        </p>
        <p style={{ margin: '4px 0' }}>
          <strong>Видимых строк:</strong> {virtualRows.length}
        </p>
        <p style={{ margin: '4px 0' }}>
          <strong>Первая видимая:</strong> индекс {virtualRows[0]?.index ?? 'нет'}
        </p>
        <p style={{ margin: '4px 0' }}>
          <strong>Последняя видимая:</strong> индекс {virtualRows[virtualRows.length - 1]?.index ?? 'нет'}
        </p>
      </div>
    </div>
  );
}
```
