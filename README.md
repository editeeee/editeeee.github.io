# editeeee.github.io
import Papa from 'papaparse';
import _ from 'lodash';

export interface ColumnStats {
  name: string;
  type: 'Text' | 'Integer' | 'Decimal' | 'Date' | 'Boolean' | 'Email';
  missing: number;
  missingPercentage: number;
  unique: number;
  outliers: number;
  mean?: number;
  median?: number;
  min?: number;
  max?: number;
  q1?: number;
  q3?: number;
  isConstant: boolean;
  isEmpty: boolean;
  inconsistentValues: string[];
}

export interface Issue {
  id: string;
  severity: 'critical' | 'warning' | 'info';
  category: string;
  message: string;
  column?: string;
}

export interface AnalysisResult {
  fileName: string;
  rowCount: number;
  colCount: number;
  totalMissing: number;
  duplicateRows: number;
  score: number;
  columns: ColumnStats[];
  issues: Issue[];
  data: any[];
}

const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;

export const analyzeCSV = (file: File): Promise<AnalysisResult> => {
  return new Promise((resolve, reject) => {
    Papa.parse(file, {
      header: true,
      dynamicTyping: true,
      skipEmptyLines: true,
      complete: (results) => {
        const data = results.data as any[];
        const headers = results.meta.fields || [];
        const rowCount = data.length;
        const columnStats: ColumnStats[] = headers.map(col => {
          const values = data.map(row => row[col]).filter(v => v !== null && v !== undefined && v !== '');
          const missing = rowCount - values.length;
          const uniqueValues = new Set(values);
          // Type Inference
          let type: ColumnStats['type'] = 'Text';
          if (values.length > 0) {
            const firstVal = values[0];
            if (typeof firstVal === 'number') {
              type = Number.isInteger(firstVal) ? 'Integer' : 'Decimal';
            } else if (typeof firstVal === 'boolean') {
              type = 'Boolean';
            } else if (!isNaN(Date.parse(firstVal)) && isNaN(Number(firstVal))) {
              type = 'Date';
            } else if (emailRegex.test(String(firstVal))) {
              type = 'Email';
            }
          }
          // Statistics for numbers
          let stats: any = {};
          if (type === 'Integer' || type === 'Decimal') {
            const sorted = [...values].sort((a, b) => a - b);
            const q1 = sorted[Math.floor(sorted.length * 0.25)];
            const q3 = sorted[Math.floor(sorted.length * 0.75)];
            const iqr = q3 - q1;
            const lowerBound = q1 - 1.5 * iqr;
            const upperBound = q3 + 1.5 * iqr;
            const outliers = values.filter(v => v < lowerBound || v > upperBound).length;
            stats = {
              min: _.min(values),
              max: _.max(values),
              mean: _.mean(values),
              median: sorted[Math.floor(sorted.length / 2)],
              q1,
              q3,
              outliers
            };
          }
          // Inconsistency check (case sensitivity/spacing)
          const inconsistencies: string[] = [];
          if (type === 'Text') {
            const normalized = values.map(v => String(v).trim().toLowerCase());
            const uniqueNormalized = new Set(normalized).size;
            if (uniqueNormalized < uniqueValues.size) {
              inconsistencies.push('Inconsistent capitalization or whitespace detected');
            }
          }
          return {
            name: col,
            type,
            missing,
            missingPercentage: (missing / rowCount) * 100,
            unique: uniqueValues.size,
            outliers: stats.outliers || 0,
            isConstant: uniqueValues.size === 1,
            isEmpty: values.length === 0,
            inconsistentValues: inconsistencies,
            ...stats
          };
        });
        // Duplicate Rows
        const stringifiedRows = data.map(r => JSON.stringify(r));
        const duplicateRows = stringifiedRows.length - new Set(stringifiedRows).size;
        // Issue Generation
        const issues: Issue[] = [];
        columnStats.forEach(col => {
          if (col.isEmpty) issues.push({ id: `empty-${col.name}`, severity: 'critical', category: 'Empty Column', message: `Column "${col.name}" is completely empty.`, column: col.name });
          if (col.missingPercentage > 10) issues.push({ id: `missing-${col.name}`, severity: 'warning', category: 'Missing Values', message: `${col.missingPercentage.toFixed(1)}% values missing in ${col.name}`, column: col.name });
          if (col.outliers > 0) issues.push({ id: `outlier-${col.name}`, severity: 'info', category: 'Outliers', message: `${col.outliers} potential outliers in ${col.name}`, column: col.name });
          if (col.inconsistentValues.length > 0) issues.push({ id: `inc-${col.name}`, severity: 'warning', category: 'Formatting', message: `Inconsistent formatting in ${col.name}`, column: col.name });
        });
        if (duplicateRows > 0) issues.push({ id: 'dupes', severity: 'warning', category: 'Duplicates', message: `${duplicateRows} duplicate rows found.` });
        // Scoring Logic
        let score = 100;
        score -= (duplicateRows > 0 ? 5 : 0);
        columnStats.forEach(c => {
          if (c.isEmpty) score -= 10;
          if (c.missingPercentage > 0) score -= Math.min(c.missingPercentage / 2, 15);
          if (c.inconsistentValues.length > 0) score -= 5;
        });
        resolve({
          fileName: file.name,
          rowCount,
          colCount: headers.length,
          totalMissing: _.sumBy(columnStats, 'missing'),
          duplicateRows,
          score: Math.max(0, Math.round(score)),
          columns: columnStats,
          issues,
          data
        });
      },
      error: (err) => reject(err)
    });
  });
};
