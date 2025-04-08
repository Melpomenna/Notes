   #Asm 
   https://stackoverflow.com/questions/46521694/what-are-mm-prefetch-locality-hints
   
   инструкция prefetch подгружает данные в кеш, в зависимости от локальности (prefetch{local}, local - локальность, подгрузка в кеш - local - уровень кеша формально), больше служит подсказкой для процессора, нежели точным указанием