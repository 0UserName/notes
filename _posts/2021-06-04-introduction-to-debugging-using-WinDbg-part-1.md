---
layout: post
title: "Introduction to debugging using WinDbg. Part 1 (v0.1.0)"
date: 2021-06-04 01:30:50 +0300
categories:
---



<h1>Introduction</h1>
<br/>
<div align="justify">
    <p>
        WinDbg — многоцелевой отладчик для операционный системы Windows, поставляемый в рамках Windows SDK.
    </p>
</div>
<br/>
<h2>Debug scope</h2>
<br/>
<div align="justify">
    <p>
        WinDbg предоставляет возможность отладки процессов пользовательского режима и режиме ядра:
    </p>
</div>
<br/>
<div align="center">
    <table>
        <caption>Table 1. Generalized comparison of <b>user-mode</b> and <b>kernel-mode</b></caption>
        <thead>
            <tr>
                <th><strong>Mode</strong></th>
                <th><strong>Priority</strong></th>
                <th><strong>Privileges</strong></th>
                <th><strong>Virtual address space</strong></th>
                <th><strong>Hardware Access</strong></th>
            </tr>
        </thead>
        <tbody>
            <tr>
                <td>
                    <p>
                        User
                    </p>
                </td>
                <td>
                    <p>
                        Низкий
                    </p>
                </td>
                <td>
                    <p>
                        Низкий
                    </p>
                </td>
                <td>
                    <p>
                        Изолированное — процессы не могут изменять данные друг друга
                    </p>
                </td>
                <td>
                    <p>
                        Запрещен
                    </p>
                </td>
            </tr>
            <tr>
                <td>
                    <p>
                        Kernel
                    </p>
                </td>
                <td>
                    <p>
                        Высокий
                    </p>
                </td>
                <td>
                    <p>
                        Высокий
                    </p>
                </td>
                <td>
                    <p>
                        Единое адресное пространство - запись по ошибочному адресу может привести к конфликтам
                    </p>
                </td>
                <td>
                    <p>
                        Разрешен
                    </p>
                </td>
            </tr>
        </tbody>
    </table>
</div>
<br/>
<h3>User-Mode</h3>
<br/>
<div align="center">
    <img src="{{site.baseurl}}/img/native_user_mode_debugging.png" alt="Native user-mode debugging architecture in Windows"/>
    <p>
        Pic 1. Native user-mode debugging architecture in Windows
    </p>
</div>
<br/>
<div align="justify">
    <p>
        Изображение, представленное выше, иллюстрирует основные принципы заложенные в архитектуру отладки:
    </p>
</div>
<br/>
<div align="justify">
    <ul>
        <li>
            При возникновении событий отладки, таких как загрузка модуля, возникновение исключения происходящих в рамках отлаживаемого процесса, ОС генерирует сообщения от имени процесса и отправляет его отладчику. В момент каждого уведомления процесс блокируется в ожидании ответа отладчика и только после этого возобновляет свое выполнение;
        </li>
        <br/>
        <li>
            Для поддержки описанного поведения, отладчик должен иметь выделенный поток для приема и ответа на события отладки, генерируемые целевым процессом;
        </li>
        <br/>
        <li>
            Межпроцессное взаимодействие (IPC) между двумя программами основано на объекте ядра debug port, в который целевой процесс ставит свои уведомления в очередь и ожидает ответа отладчика.
        </li>
    </ul>
</div>
<br/>
<h2>Debug mode</h2>
<br/>
<h3>In-Process</h3>
<br/>
<div align="justify">
    Подразумевает наличие действующего, целевого процесса, с которым отладчик связывается одним из следующих способов:
</div>
<br/>
<div align="justify">
    <ul>
        <li>
            <a href="https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/noninvasive-debugging--user-mode-">Noninvasive</a> : отладчик открывает процесс функцией <a href="https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-openprocess">OpenProcess</a>, попутно приостанавливая все потоки. Режим предусматривает доступ к памяти, регистрам и прочей информации, однако контролировать процесс в данном случае невозможно, например, нельзя назначить точку останова. Полезен, когда процесс перестал отвечать;
        </li>
        <br/>
        <li>
            Invasive: для установления связи между отладчиком и процессом используется функция <a href="https://docs.microsoft.com/en-us/windows/win32/api/debugapi/nf-debugapi-debugactiveprocess">DebugActiveProcess</a> (если не указано обратного). Режим предусматривает доступ к памяти, регистрам. Позволяет использовать весь набор отладочных команд: назначать точки останова, возобновлять выполнение процесса/потока и т.д.
        </li>
    </ul>
</div>
<br/>
<h3>Out-of-process</h3>
<br/>
<div align="justify">
    <p>
        Не подразумевает наличия действующего, целевого процесса, используется при анализе дампов. Никаких подтипов у данного режима нет, однако стоит отметить, что при создании дампа важно учитывать битность процесса - для 32bit/64bit процесса должен быть создан 32bit/64bit дамп. В противном случае могут возникнуть сложности с получением отладочной информации - это вызвано тем, что 32bit процесс запускается в 64bit ОС через специальный слой совместимости - <a href="https://www.fireeye.com/blog/threat-research/2020/11/wow64-subsystem-internals-and-hooking-techniques.html">WoW64</a>. 
    </p>
</div>
<br/>
<h3>WoW64</h3>
<br/>
<div align="justify">
    <p>
        Данная подсистема перехватывает системные (syscall) вызовы из 32bit версий <b>ntdll.dll</b> и <b>user32.dll</b> от имени 32bit приложений и преобразует их в 64bit вызовы, конвертирует аргументы так, чтобы размер указателей и стек фреймы были корректными. Этот процесс называется маршалингом, реализован в <b>wow64.dll</b> для <b>ntdll.dll</b> и в <b>wow64win.dll</b> для GDI/USER32, которые в свою очередь используют <b>wow64cpu.dll</b> и в которой реализованы 64 - битные инструкции переключения режима на языке ассемблера. 
    </p>
</div>
<br/>
<div align="center">
    <img src="{{site.baseurl}}/img/wow64_0.png" alt="Output of the kc 64bit dump of a 32bit process before and after switching to WoW mode"/>
    <p> 
        Pic 2. Output of the kc 64bit dump of a 32bit process before and after switching to WoW mode
    </p>
</div>
<br/>
<div align="justify">
    <p>
        В первом случае, в стеке отображаются методы <a href="https://www.fireeye.com/blog/threat-research/2020/11/wow64-subsystem-internals-and-hooking-techniques.html">WoW64</a>, в то время как во втором отображается относительно корректный стек. Переключение в WoW режим выполнен с помощью расширения <a href="https://github.com/poizan42/soswow64">soswow64</a>.
    </p>
</div>
<br/>
<div align="center">
    <img src="{{site.baseurl}}/img/wow64_1.png" alt="Pic 3. WOW64 architecture overview"/>
    <p>
        Pic 3. WOW64 architecture overview 
    </p>
</div>
<br/>
<h2>Commands</h2>
<br/>
<div align="center">
    <table>
        <caption>Table 2. Types of commands presented in WinDbg</caption>
        <thead>
            <tr>
                <th>
                    <strong>
                        <p>
                            Type
                        </p>
                    </strong>
                </th>
                <th>
                    <strong>
                        <p>
                            Description
                        </p>
                    </strong>
                </th>
                <th>
                    <strong>
                        <p>
                            Examples
                        </p>
                    </strong>
                </th>
                <th>
                    <strong>
                        <p>
                            Note
                        </p>
                    </strong>
                </th>
            </tr>
        </thead>
        <tbody>
            <tr>
                <td>
                    <p>
                        <a href="https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/debugger-commands">Debugger</a>
                    </p>
                </td>
                <td>
                    <p>
                        Basic debugger commands
                    </p>
                </td>
                <td>
                    <p>
                        d, da, db, dc, dd, dD, df, dp, dq, du, dw, dW, dyb, dyd (Display Memory)
                    </p>
                    <p>
                        dda, ddp, ddu, dpa, dpp, dpu, dqa, dqp, dqu (Display Referenced Memory) 
                    </p>
                    <p>
                        dds, dps, dqs (Display Words and Symbols)
                    </p>
                    <p>
                        k, kb, kc, kd, kp, kP, kv (Display Stack Backtrace)
                    </p>
                    <p>
                        ld (Load Symbols)
                    </p>
                    <p>
                        lm (List Loaded Modules)
                    </p>
                    <p>
                        r (Registers)
                    </p>
                    <p>
                        s (Search Memory)
                    </p>
                    <p>
                        x (Examine Symbols)
                    </p>
                    <p>
                        ~e (Thread-Specific Command)
                    </p>
                </td>
                <td>
                    <p>
                        The last characters regulate the data format, for example:
                    </p>
                    <p>
                        dd – Double-word values dq – Quad-word values
                    </p>
                    <p>
                        da – ASCII characters du – Unicode characters
                    </p>
                </td>
            </tr>
            <tr>
                <td>
                    <p>
                        <a href="https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/meta-commands">Meta (dot)</a>
                    </p>
                </td>
                <td>
                    <p>
                        Commands that control debugger behavior
                    </p>
                </td>
                <td>
                    <p>
                        .load, .loadby (Load Extension DLL) .logopen (Open Log File)
                    </p>
                    <p>
                        .logclose (Close Log File)
                    </p>
                    <p>
                        .symfix (Set Symbol Store Path) .sympath (Set Symbol Path)
                    </p>
                </td>
                <td>
                    <p>
                        Start with "."
                    </p>
                </td>
            </tr>
            <tr>
                <td><a href="https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/general-extensions"> Extension</a></td>
                <td>
                    <p>
                        Commands exported by extensions
                    </p>
                </td>
                <td>
                    <p>
                        <a href="https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/-locks---ntsdexts-locks-">!locks (!ntsdexts.locks)</a> (Displays a list of critical sections associated with the current process)
                    </p>
                    <p>
                        <a href="https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/-uniqstack">!uniqstack</a> (Displays all of the stacks for all of the threads in the current process, excluding stacks that appear to have duplicates)
                    </p>
                </td>
                <td>
                    <p>
                        Start with "!"
                    </p>
                </td>
            </tr>
        </tbody>
    </table>
</div>
<h2>Scripting</h2>
<br/>
<div align="justify">
    <p>
        WinDbg имеет встроенный механизм выполнения сценариев, который позволяет автоматизировать рутинные действия. В качестве языковых элементов используются <a href="https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/command-tokens">Command Tokens</a>, которые очень похоже на конструкции в привычных языках программирования, таких как C/C++:
    </p>
</div>
<br/>
<div align="center">
    <table>
        <caption>Table 3. Control tokens by groups</caption>
        <thead>
            <tr>
                <th><strong>Token</strong></th>
                <th><strong>Category</strong></th>
            </tr>
        </thead>
        <tbody>
            <tr>
                <td>
                    <p>
                        <a href="https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/----command-separator-">(Command Separator)</a>; <a href="https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/-block">.block</a>; <a href="https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/------block-delimiter-">{ } (Block Delimiter)</a>
                    </p>
                </td>
                <td>
                    <p>
                        Delimiters
                    </p>
                </td>
            </tr>
            <tr>
                <td>
                    <p>
                        <a href="https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/-----comment-specifier-">$$ (Comment Specifier)</a>; <a href="https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/----comment-line-specifier-">* (Comment Line Specifier)</a>
                    </p>
                </td>
                <td>
                    <p>
                        Comments
                    </p>
                </td>
            </tr>
            <tr>
                <td>
                    <p>
                        <a href="https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/-printf">.printf</a>
                    </p>
                </td>
                <td>
                    <p>
                        Output
                    </p>
                </td>
            </tr>
            <tr>
                <td>
                    <p>
                        <a href="https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/-if">.if</a>; <a href="https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/-else">.else</a>; <a href="https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/-elsif">.elsif</a>
                    </p>
                </td>
                <td>
                    <p>
                        Logical conditions
                    </p>
                </td>
            </tr>
            <tr>
                <td>
                    <p>
                        <a href="https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/-for">.for</a>; <a href="https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/-foreach">.foreach</a>; <a href="https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/-do">.do</a>; <a href="https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/-while">.while</a>
                    </p>
                </td>
                <td>
                    <p>
                        Loops
                    </p>
                </td>
            </tr>
            <tr>
                <td>
                    <p>
                        <a href="https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/-continue">.continue</a>; <a href="https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/-break3">.break</a>
                    </p>
                </td>
                <td>
                    <p>
                        Jump statement
                    </p>
                </td>
            </tr>
            <tr>
                <td>
                    <p>
                        <a href="https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/-catch">.catch</a>; <a href="https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/-leave">.leave</a>
                    </p>
                </td>
                <td>
                    <p>
                        Exceptions
                    </p>
                </td>
            </tr>
        </tbody>
    </table>
</div>
<br/>
<div align="justify">
    <p>
        Комбинация перечисленных токенов позволяет составлять произвольные сценарии, примерная ситуация: в логах SQL Server после его аварийного завершения была обнаружена следующая запись -
    </p>
    <br/>
    <p>
        <q>
        <i>
        A time-out occurred while waiting for buffer latch -- type 4, bp 00000208B31DF840, page 3:227674, stat 0x10b, database id: 2, allocation unit Id: 562949955649536, task 0x0000021BA22784E8: 0, waittime 300 seconds, flags 0x3a, owning task <b>0x000002254F58F468</b>. Not continuing to wait.
        </i>
        </q>
    </p>
    <br/>
    <p>
        Требуется найти идентификатор потока, который содержит в памяти следующий байтовый паттерн: <b>0x000002254F58F468</b>:
    </p>
    <br/>
    <code>
    ~*e .echo ThreadId:; ?? @$tid; r? @$t1 = ((ntdll!_NT_TIB*)@$teb)->StackLimit; r? @$t2 = ((ntdll!_NT_TIB)@$teb)->StackBase; s -q @$t1 @$t2 0x000002254F58F468
    </code>
</div>
<br/>
<h2>Extensions (for .NET)</h2>
<br/>
<div align="justify">
    <p>
        WinDbg - нативный отладчик, по этой причине ему ничего не известно об управляемом коде и он не может самостоятельно интерпретировать его структуры. Для решения это проблемы необходимо использовать <a href="https://docs.microsoft.com/en-us/dotnet/framework/tools/sos-dll-sos-debugging-extension"> SOS (Son of Strike)</a> расширение:
        <br/>
        <br/>
        <q>
        <i>
        The SOS Debugging Extension (SOS.dll) helps you debug managed programs in Visual Studio and in the Windows debugger (WinDbg.exe) by providing information about the internal Common Language Runtime (CLR) environment. 
        </i>
        </q>
    </p>
</div>
<br/>
<div align="center">
    <table>
        <caption>Table 4. External extensions for debugging managed code</caption>
        <thead>
            <tr>
                <th><strong>Name</strong></th>
                <th><strong>SOS required</strong></th>
                <th><strong>Description</strong></th>
            </tr>
        </thead>
        <tbody>
            <tr>
                <td><a href="https://github.com/poizan42/soswow64">soswow64</a></td>
                <td>Y</td>
                <td>This extension gets around this by hooking/patching functions in dbgeng.dll so that SoS thinks it's working with a 32-bit dump.</td>
            </tr>
            <tr>
                <td><a href="https://github.com/rodneyviana/netext">netex</a></td>
                <td>N</td>
                <td>WinDbg extension for data mining managed heap. It also includes commands to list HTTP request, WCF services, WIF tokens among others.</td>
            </tr>
            <tr>
                <td><a href="https://github.com/microsoft/vs-threading/blob/main/doc/dumpasync.md">dumpasync</a></td>
                <td>Y</td>
                <td>WinDbg extension that produces a list of async method callstacks that may not appear on thread callstacks. This information is useful when diagnosing the cause for an app hang that is waiting for async methods to complete.</td>
            </tr>
            <tr>
                <td><a href="https://github.com/REhints/WinDbg/tree/master/MEX">MEX</a></td>
                <td>?</td>
                <td>Extension for WinDbg can help you simplify common debugger tasks, and provides powerful text filtering capabilities to the debugger.</td>
            </tr>
        </tbody>
    </table>
</div>
<br/>
<div align="justify">
    <p>
        В данном разделе перечислены только внешние расширения, которые автор счел полезными. WinDbg имеет в своем арсенале значительный набор встроенных расширений общего назначения, работающих как режиме пользователя (user-mode) так и ядра (kernel mode). Кроме того, имеются <a href="https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/specialized-extensions">специализированные</a> расширения драйверов (USB, Bluetooth) и т.п. 
    </p>
</div>