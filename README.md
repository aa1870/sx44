@echo off
REM —— 防止中文乱码（UTF‑8 BOM）  
chcp 65001 >nul  
title 自动监控剪贴板 → 提取链接

setlocal enabledelayedexpansion

REM 临时文件定义  
set "lastClip=%~dp0_last_clip.tmp"  
set "curClip=%~dp0_cur_clip.tmp"  
set "tempLinks=%~dp0_links.tmp.txt"

REM 如果有旧的记录文件，删掉  
if exist "%lastClip%" del /f /q "%lastClip%"

:monitor
    REM —— 每秒读取剪贴板内容到 cur_clip  
    powershell -NoProfile -Command "Get-Clipboard -Raw | Out-File -Encoding UTF8 '%curClip%'"  

    REM —— 比对和上次内容是否相同  
    fc "%curClip%" "%lastClip%" >nul 2>&1  
    if errorlevel 1 (
        REM —— 内容变了，开始处理  
        call :process "%curClip%"

        REM —— 更新 last_clip  
        copy /y "%curClip%" "%lastClip%" >nul
    )

    REM —— 等待 1 秒再重试  
    timeout /t 1 /nobreak >nul
goto monitor

:process
    set "infile=%~1"

    echo.
    echo [*] 检测到新剪贴板内容，正在提取链接…

    REM —— 删除旧 .txt 文件  
    del /f /q "%~dp0*.txt" >nul 2>&1

    REM —— 用 PowerShell 提取去重链接到临时文件  
    powershell -NoProfile -Command ^
      "$txt = Get-Content -LiteralPath '%infile%' -Raw; " ^
      "[regex]::Matches($txt,'(https?://[^\s\)\]]+)|(t\.me/[^\s\)\]]+)|(telegram\.me/[^\s\)\]]+)|(ftp://[^\s\)\]]+)|(ftps://[^\s\)\]]+)|(www\.[^\s\)\]]+)') " ^
      "| ForEach-Object{ $_.Groups | ForEach-Object{ $_.Value } } " ^
      "| Where-Object{ $_ } | Sort-Object -Unique " ^
      "| Out-File -Encoding UTF8 -FilePath '%tempLinks%'"

    REM —— 统计链接数量  
    for /f %%N in ('find /v /c "" ^< "%tempLinks%"') do set "count=%%N"

    REM —— 构造输出文件名：数量_链接.txt  
    set "outfile=%~dp0%count%_链接.txt"

    REM —— 写入提示行＋链接列表  
    (echo 共提取链接 %count% 条：) > "%outfile%"  
    type "%tempLinks%" >> "%outfile%"

    REM —— 清理临时文件  
    del /f /q "%tempLinks%" >nul

    echo [√] 已保存：%outfile%
    echo.

    exit /b
