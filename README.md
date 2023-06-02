[ https://htmlpreview.github.io/?https://github.com/Nitr0-G/Reverse-engineering-a-signed-kernel-driver-ft.-VMProtect-RUS/blob/main/README.html](https://htmlpreview.github.io/?https://github.com/Nitr0-G/Reverse-engineering-a-signed-kernel-driver-ft.-VMProtect-RUS/blob/main/README.html)

[zer0condition](https://zerocondition.com/)

*   [About](https://zerocondition.com/about/)
*   [GitHub](https://github.com/zer0condition)
*   [Instagram](https://www.instagram.com/zer0condition/)

*   [About](https://zerocondition.com/about/)
*   [GitHub](https://github.com/zer0condition)
*   [Instagram](https://www.instagram.com/zer0condition/)

Реверсим signed kernel driver ft. VMProtect. Перевел Nitr0-G.
===========================================

14 April, 2023 — 20 min read

Начало начал:
-------------

Недавно ко мне обратился пользователь discord, который попросил меня взломать платный чит-сервис, название которого я не буду называть, поскольку он был забанен ими после того, как испытал некоторые трудности при попытке воспользоваться их услугами. Хотя поначалу я колебался, но скука взяла надо мной верх, и я решил взглянуть. Я не был удивлен, обнаружив, что загрузчик был упакован и виртуализирован с использованием одного из новейших **VMProtect3**версии, которые немного усложняли ситуацию. Однако с помощью динамического анализа я обнаружил, что двоичный файл был на диске в моем C:\\Windows\\System32. Я ожидал, что это будет исполняемый файл, но он имел расширение .sys и вызывался 'winhelper.sys '. При ближайшем рассмотрении я обнаружил, что размер драйвера составляет около 2 МБ, что необычно для дриверов, а его цифровой сертификат был подписан revoked/expired сертификатом EV от китайской компании под названием **'Binzhoushi Yongyu Feed Co.,LTd.'**

![deviceo](https://i.imgur.com/J8wchy4.png)

Расследование:
--------------

Дальнейшее расследование показало, что driver's timestamp относится к 2015 году, что было необычно. Я решил загрузить драйвер в IDA и обнаружил, что он был упакован и виртуализирован с помощью VMProtect 3 еще раз.

![deviceo](https://i.imgur.com/YNXvkUv.png)

К сожалению, EP была виртуализирована, поэтому мне пришлось прибегнуть к помощи друга, чтобы девиртнуть файл с помощью тулзов. Получив девиртуализированный файл, я углубился во внутренние устройства драйвера и обнаружил, что он использует I/O для communication, что не является чем-то необычным.

В конце концов, я нашел driver object's reference, выполнил подфункцию и нашел обработчик IRP/Dispatch драйвера.

![deviceo](https://i.imgur.com/9KGoS40.png) ![device1](https://i.imgur.com/KxgaJws.png)

Хотя функциональность была не слишком сложной, управляющий код, передаваемый через расположение стека, был "зашифрован" с помощью некоторых операций XOR и bitwise операций. Пример программы driver controller объясняет, как это работает.

![deviceo](https://i.imgur.com/wf64H8f.png)

IOCTLs and Functionalities:
---------------------------

Я решил ознакомиться с функциональными возможностями драйверов и их control codes.

#### GetProcessBaseAddresss:

**0x13370400**: Это был первый код ioctl, с которым я столкнулся, который использует структуру с двумя переменными, отправляемыми через системный буфер IRP. Буфер возвращается в user mode после выполнения реквеста. Он принимает интовый параметр process id, который используется для PsLookupProcessByProcessId() для получения EPROCESS целевого процесса и передается в PsGetProcessSectionBaseAddress(), который возвращает SectionBaseAddress EPROCESS. Затем доступ к базовому адресу осуществляется из user mode со второй переменной внутри структуры.

![deviceo](https://i.imgur.com/aoIwRT3.png)

#### ReadProcessMemory:

**0x13370800**: Это был второй найденный код ioctl, он также использует структуру, переданную из SystemBuffer, но другого размера, содержащую int, uintptr\_t, uintptr\_t и size\_t соответственно. Позже выяснилось, что первым параметром является идентификатор процесса, переданный из запроса usermode, вторым - адрес источника, третьим - буфер и, наконец, размер.

![deviceo](https://i.imgur.com/jevpmlw.png)

Дальнейший анализ выявил назначение двух приведенных здесь вызовов функций. Первый вызов функции настраивает функции, необходимые для процесса чтения/записи в память:

![deviceo](https://i.imgur.com/AUBnFlN.png) ![device1](https://i.imgur.com/fH2jbOh.png)Второй вызов функции считывает память процесса через физическую память. Он принимает source address, buffer, size, and a variable, которая возвращает значение, но больше не используется:

![deviceo](https://i.imgur.com/q0O8EwJ.png)

Заглянув внутрь функции, можно увидеть, что в ней нет ничего такого сложного; она принимает переданный виртуальный адрес и преобразует его в физический адрес (линейное преобразование) и используется в MmCopyMemory для чтения памяти процесса.

![deviceo](https://i.imgur.com/ozft12C.png)

#### WriteProcessMemory

**0x13370C00**: Это был последний найденный код, предназначенный для записи в память процесса. Он принимает структуру того же размера, что и readprocessmemory, с теми же переменными. Та же функция, вызванная ранее в обработчике запроса на чтение, настраивает функцию записи, которая будет использоваться, и сама функция записи вызывается после этого, принимая адрес источника, буфер, размер и переменную, которая возвращает значение, но больше не используется.

![deviceo](https://i.imgur.com/hcenajT.png)

Функция write принимает переданный виртуальный адрес и преобразует его в физический адрес (линейное преобразование) и используется в MmMapIoSpaceEx для записи/отображения значений в память процесса.

![deviceo](https://i.imgur.com/rYDFhhl.png) ![deviceo](https://i.imgur.com/sbVJfYw.png)

Вывод:
------

В целом, реверс signed kernel driver был интересной и сложной задачей. Это включало динамический анализ, девиртуализацию и исследование driver's internals, чтобы раскрыть его функциональные возможности и control codes.

Если копнуть глубже, причина, по которой этот отозванный сертификат мог быть загружен, приводит к китайскому инструменту под названием "HookSigntool".

*   [HookSignTool](https://github.com/Jemmy1228/HookSigntool)

Ссылка на репозиторий содержит подписанный двоичный файл вместе с примером программы о том, как отправлять запросы к этому драйверу и технически использовать его.

*   [GitHub Repository](https://www.github.com/zer0condition/Reversing-a-signed-driver)

Read other posts

* * *

[Demystifying PatchGuard: An In-Depth Analysis Through Practical Engineering →](https://zerocondition.com/posts/demystifying-patchguard/)

© 2023 zerocondition  
contact@zerocondition.com

 Posts :: Reverse engineering a signed kernel driver ft. VMProtect
